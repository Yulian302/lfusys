# LFUSYS

A microservices-based large file upload system with a React frontend.
The repository contains backend Go services (gateway, sessions, uploads, shared commons), Terraform infra, and a Vite + React frontend.

## Quick Summary
- Language: **Go** (backend services using **Gin** and **gRPC**) and **TypeScript/React** (frontend).
- Services: `gateway`, `sessions`, `uploads`, `commons` (shared libs).
- Infra: Terraform modules (**DynamoDB**, **S3**, **SQS**, other mandatory cloud and security services).
- Dev: **Docker Compose** for local development.

## Contents
- backend/: Go services, infra, modules, shared code
- frontend/: Vite + React app (`lfusys-app`)
- docker-compose.yml, Makefiles and per-service Dockerfiles

## How it works (high-level)
- The React frontend talks to the `gateway` service for auth and uploads session creation.
- `gateway` implements HTTP routes and delegates business logic to internal service implementations (AuthService, UploadsService). `gateway` talks to `session` service via **gRPC** to create upload session.
- `session` service persists session metadata (DynamoDB) and manages the status of upload.

**At the same time**
- Frontend breaks the whole file into chunks and upload them to `upload` service workers in parallel.
- `upload` service consists of multiple workers that validate the upload integrity, persist uploaded chunks to AWS S3 object store and update the status of upload.
- When upload is complete, last `upload` worker puts the session ID into the distributed FIFO queue (AWS SQS).
- `session` service consumes the upload sessions from the queue and creates a final `File` object using assembly-based approach for contents storage.

## Authentication
JWT authentication is implemented under the hood with the best security practices.

# Architecture
This section provides the diagrams for different layers of abstration in the system. It also includes design scratches `as is` to show how system changed and evolved. Initial diagrams may have flaws and may be overly opinionated, but it greatly depicts the engineer's thinking process and how can one come to a better architecture over time.

### Initial High-Level Design Sketch
![Initial Design Sketch](https://github.com/Yulian302/lfusys-infra/blob/196796faccfee576463794a84295194094da6d1e/design/design.png)

## Evolved Service Interactions Diagram
```mermaid

flowchart LR
    FE["React Frontend"] <-- HTTP --> GW["Gateway"]
    GW <-- gRPC --> US["Sessions Service"]
    US --> DDB1[("DynamoDB Uploads")]
    GW --> DDB2[("DynamoDB Users")]
    FE <-- HTTP --> UW["Upload Service"]
    UW --> S3[("S3 Object Storage")]
```

## Sequence Diagrams
Here are a few sequence diagrams that represent core auth and business logic.
### Login


```mermaid

sequenceDiagram
    participant FE as Frontend
    participant GW as Gateway Service (Auth + API)
    participant DB@{"type":"database"}

    FE->>GW: POST /auth/login (email, password)
    activate GW

    GW->>GW: Validate request payload
    alt Invalid payload
        GW-->>FE: 400 Bad Request
        deactivate GW

    else Valid payload
        activate GW
        GW->>DB: Fetch user by email
        activate DB

        alt User not found
            DB-->>GW: No record
            deactivate DB
            GW->>FE: 401 Unauthorized
            deactivate GW

        else User found
            activate DB
            activate GW
            DB-->>GW: User data
            deactivate DB
            GW->>GW: Verify password hash

            alt Password incorrect
                GW-->>FE: 401 Unauthorized
                deactivate GW
            else Password correct
                activate GW
                GW->>GW: Create JWT access + refresh tokens
                GW-->>FE: Set-Cookie (access, refresh)
                GW-->>FE: 200 OK
                deactivate GW
            end
        end
    end

```

### Register

```mermaid

sequenceDiagram
    participant FE as Frontend
    participant GW as Gateway Service (Auth + API)
    participant DB@{"type":"database"}

    FE->>GW: POST /auth/register (name, email, password)
    activate GW

    GW->>GW: Validate request payload
    alt Invalid payload
        GW-->>FE: 400 Bad Request
        deactivate GW

    else Valid payload
        activate GW
        GW->>GW: Hash password with salt
        GW->>DB: Create user (salt, hash)
        activate DB

        alt User already exists
            DB-->>GW: Error
            deactivate DB
            GW->>FE: 409 Status Conflict
            deactivate GW

        else Internal error
            activate GW
            activate DB
            DB-->>GW: Error
            deactivate DB
            GW->>FE: 500 Internal Server Error
            deactivate GW

        else User does not exist
            activate DB
            activate GW
            DB-->>GW: User id
            deactivate DB
            GW-->>FE: 201 Created
            deactivate GW
        end
    end

```

### Refresh Token
```mermaid

sequenceDiagram
    participant FE as Frontend
    participant GW as Gateway Service (Auth + API)
    participant DB@{"type":"database"}

    FE->>GW: GET /auth/refresh (refresh_token)
    activate GW

    GW->>GW: Validate refresh token
    alt Missing token
        GW-->>FE: 401 Unauthorized
        deactivate GW

    else Invalid token
        activate GW
        GW->>GW: Parse token
        GW-->>FE: 401 Unauthorized
        deactivate GW

    else Wrong type
        activate GW
        GW->>FE: 401 Unauthorized
        deactivate GW

    else Valid token
        GW->>DB: Get user (email)
        activate DB
    end

    alt User inactive or not found
        activate GW
        DB-->>GW: Error
        deactivate DB
        GW-->>FE: 401 Unauthorized
        deactivate GW

    else User exists & active
        activate DB
        DB-->>GW: User exists
        deactivate DB
        activate GW
        GW->>GW: Create new access & refresh token

        alt Token fails to create
            GW->>FE: 500 Internal Server Error
            deactivate GW

        else Token created
            activate GW
            GW-->>FE: 200 OK (HTTPOnly access, refresh)
            deactivate GW
        end

    end

```

### Me (User Profile)

```mermaid

sequenceDiagram
    participant FE as Frontend
    participant GW as Gateway Service (Auth + API)
    participant DB@{"type":"database"}

    FE->>GW: GET /auth/me (jwt)
    activate GW

    GW->>GW: Validate jwt
    alt Missing token
        GW-->>FE: 401 Unauthorized
        deactivate GW

    else Invalid token
        activate GW

        GW->>GW: Parse token
        GW-->>FE: 401 Unauthorized
        deactivate GW

    else Valid token
        activate GW
        GW->>DB: Get user (email)
        activate DB

        alt Internal Error
            DB-->>GW: Error
            deactivate DB
            GW->>FE: 500 Internal Server Error
            deactivate GW

        else Query success
            activate GW
            activate DB
            DB-->>GW: User data
            deactivate DB
            GW-->>FE: 200 OK (User)
            deactivate GW
        end
    end

```

## Contact
[Telegram](https://t.me/julianoadmin)
