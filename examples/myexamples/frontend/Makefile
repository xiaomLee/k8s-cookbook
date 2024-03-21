IMAGE_REGISTRY ?= docker.io
IMAGE_REGISTRY_USER ?=
IMAGE_REGISTRY_PASS ?=
VERSION ?= $(shell git symbolic-ref --short head)-$(shell git rev-parse --short HEAD)
IMAGE_REPO_BASE ?= ${IMAGE_REGISTRY}/850278995

PROJECT ?= frontend

docker-login:
	docker login ${IMAGE_REGISTRY} -u'${IMAGE_REGISTRY_USER}' -p'${IMAGE_REGISTRY_PASS}'

image:
	docker build -t ${IMAGE_REPO_BASE}/${PROJECT}:${VERSION} .
	docker tag ${IMAGE_REPO_BASE}/${PROJECT}:${VERSION} ${IMAGE_REPO_BASE}/${PROJECT}:latest
	docker push ${IMAGE_REPO_BASE}/${PROJECT}:${VERSION}
	docker push ${IMAGE_REPO_BASE}/${PROJECT}:latest