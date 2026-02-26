# LFUSYS

A microservices-based large file upload system with a React frontend.
The repository contains backend Go services (gateway, sessions, uploads, shared commons), Terraform infra, and a Vite + React frontend.

## Quick Summary
- Language: **Go** (backend services using **Gin** and **gRPC**) and **TypeScript/React** (frontend).
- Services: `gateway`, `sessions`, `uploads`, `commons` (shared libs).
- Infra: Terraform modules (**DynamoDB**, **S3**, **SQS**, other mandatory cloud and security services).
- Dev: **Docker Compose** for local development.

## Preview
| | | |
|:---:|:---:|:---:|
| ![Preview1](https://github.com/Yulian302/lfusys-infra/raw/bf5509d0bced15f03195611f95edac1f256eb742/previews/preview_1.png) | ![Preview1Extra](https://github.com/Yulian302/lfusys-infra/raw/bf5509d0bced15f03195611f95edac1f256eb742/previews/preview_1_extra.png) | ![Preview2](https://github.com/Yulian302/lfusys-infra/raw/bf5509d0bced15f03195611f95edac1f256eb742/previews/preview_2.png) |
| ![Preview3](https://github.com/Yulian302/lfusys-infra/raw/bf5509d0bced15f03195611f95edac1f256eb742/previews/preview_3.png) | ![Preview4](https://github.com/Yulian302/lfusys-infra/raw/bf5509d0bced15f03195611f95edac1f256eb742/previews/preview_4.png) | ![Preview5](https://github.com/Yulian302/lfusys-infra/raw/bf5509d0bced15f03195611f95edac1f256eb742/previews/preview_5.png) |

## Deployment 🚀
### Using Docker <span style="vertical-align: middle;">![Docker](https://img.shields.io/badge/-blue?logo=docker&logoColor=white)
</span>
This project supports two deployment modes:

- **Development mode** - with tracing, verbose logging, hot reload and observability tools.
- **Production mode** - minimal images w/o development tools and containers, optimized for performance.

#### Development mode <span style="vertical-align: middle;">![dev](https://img.shields.io/badge/-white?logo=dev.to&logoColor=black&style=plastic)
</span>
Includes:

- Structured debug logging
- Comprehensive app events logging
- OpenTelemetry tracing
- Jaeger UI
- Hot reload using Air (if enabled)
- Swagger API docs
#### Prerequisites 📦
- Docker >= 24
- Docker Compose v2

#### How to run ▶️
Running from project root:
```bash
docker compose -f docker-compose.yml up --build
```
This will start:
| Service    |   | Port            |
|------------|---|-----------------|
| gateway    |   | 8080 [exposed]  |
| sessions   |   | 50051           |
| uploads    |   | 8081 [exposed]  |
| frontend   |   | 3000 [exposed]  |
| jaeger     |   | 16686 [exposed] |
| localstack |   | 4566 [exposed]  |

WebUI access locally: http://localhost:3000

#### Production mode <span style="vertical-align: middle;">![dev](https://img.shields.io/badge/-white?logo=protodotio&logoColor=black&style=plastic)
</span>

Production mode is optimized for:

- Smaller image size
- No debug tooling
- No Swagger
- Info-level event logging
- Tracing disabled (unless explicitly enabled)
- Images are built using multi-stage Docker builds.

#### How to run ▶️
You can either pull existing images from DockerHub or build them by yourself.
#### Pull images from DockerHub:
Running from project root:
```bash
docker compose -f docker-compose.prod.hub.yml up
```
OR
#### Build the images:
Running from project root:
```bash
docker compose -f docker-compose.prod.yml up --build
```
This will start:
| Service  |   | Port           |
|----------|---|----------------|
| gateway  |   | 8080 [exposed] |
| sessions |   | 50051          |
| uploads  |   | 8081 [exposed] |
| frontend |   | 80 [exposed]   |

WebUI access locally: http://localhost

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
### JWT
JWT authentication is implemented under the hood with the best security practices. It is a default authentication mechanism.
### OAuth2
Third-party authentication mechanism is enabled. Users can choose among the following providers: **Github**, **Google**.
#### OAuth2 Providers UML diagram
```mermaid
---
config:
  layout: elk
---
classDiagram
        class OAuthProvider{
            <<interface>>
            +ExchangeCode(ctx, code) (OAuthUser, error)
            +GetOAuthUser(ctx, token) (OAuthUser, error)
        }

        class GithubProvider{
            -cfg    *github.GithubConfig
	        -client *auth.Client
            +ExchangeCode(ctx, code) (OAuthUser, error)
            +GetOAuthUser(ctx, token) (OAuthUser, error)
        }

        class GoogleProvider{
            -cfg    *google.GoogleConfig
	        -client *auth.Client
            +ExchangeCode(ctx, code) (OAuthUser, error)
            +GetOAuthUser(ctx, token) (OAuthUser, error)
        }

        class Client {
            -http *http.Client
            +PostFormJSON(ctx, endpoint, data, out) error
            +GetJSONWithToken(ctx, endpoint, token, out) error
        }

        class http.Client{
            ...
        }
        class GithubConfig {
            +ClientID     string
            +ClientSecret string
            +RedirectURI  string
            +ExchangeURL  string
            +FrontendURL  string
        }

        class GoogleConfig {
            +ClientID     string
            +ClientSecret string
            +RedirectURI  string
            +ExchangeURL  string
            +FrontendURL  string
        }


        GithubProvider ..|> OAuthProvider: implements
        GoogleProvider ..|> OAuthProvider: implements

        Client --* http.Client: uses
        GithubProvider --* Client: uses
        GoogleProvider --* Client: uses

        GithubProvider --* GithubConfig: uses
        GoogleProvider --* GoogleConfig: uses
```
This architecture can be improved further, but it is an overhead for the system (no more third-party auth providers will be added).

# Architecture
This section provides the diagrams for different layers of abstration in the system. It also includes design scratches `as is` to show how system changed and evolved. Initial diagrams may have flaws and may be overly opinionated, but it greatly depicts the engineer's thinking process and how can one come to a better architecture over time.

### Initial High-Level Design Sketch
![Initial Design Sketch](https://github.com/Yulian302/lfusys-infra/blob/196796faccfee576463794a84295194094da6d1e/design/design.png)

## Resources
### Redis
| Usage               | Critical | On failure                                             |
|---------------------|----------|--------------------------------------------------------|
| Rate limit          | No       | Rate limit disabled                                    |
| Caching             | No       | Caching disabled. Responses are slower, more expensive |
| OAuth State Storage | No       | Frontend state is not stored. Small security breach    |

Redis is not critical for the system. It's a soft dependency for other components. Only **ONE** redis instance is fine for now to do all the work.

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

### UPDATED 15/01/2026
![SI diagram](https://github.com/Yulian302/lfusys-infra/blob/master/design/evolution/15-01-2026.png)


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
