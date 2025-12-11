DOCKERFILE = Dockerfile.dev
SERVICES = gateway sessions uploads

.PHONY: help
help:
	@echo "Available commands"
	@echo "	make build SERVICE=gateway	- Build specific service"
	@echo "	make run SERVICE=gateway	- Run specific service"
	@echo "	make build-all 			- Build all services"
	@echo " 	make clean				- Clean Docker images"


.PHONY: build
build:
	@if [ -z "$(SERVICE)" ]; then echo "Usage: make build SERVICE=gateway"; exit 1; fi
	docker build . --target $(SERVICE) -t lfusys-services-$(SERVICE) -f $(DOCKERFILE)


.PHONY: run
run:
	@if [ -z "$(SERVICE)" ]; then echo "Usage: make run SERVICE=gateway [PORT=8080]"; exit 1; fi
	docker run $(if $(PORT),-p $(PORT):$(PORT),) lfusys-services-$(SERVICE)

.PHONY: build-all
build-all:
	@for service in $(SERVICES); do \
		echo "Building $$service"; \
		docker build . -f $(DOCKERFILE) --target $$service -t lfusys-services-$$service; \
	done

.PHONY: compose
compose:
	DOCKER_BUILDKIT=1 docker-compose up


.PHONY: clean
clean:
	docker system prune -f 