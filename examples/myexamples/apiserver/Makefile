IMAGE_REGISTRY ?= docker.io
IMAGE_REGISTRY_USER ?=
IMAGE_REGISTRY_PASS ?=
VERSION ?= $(shell git symbolic-ref --short head)-$(shell git rev-parse --short HEAD)
IMAGE_REPO_BASE ?= ${IMAGE_REGISTRY}/850278995

OS ?= $(shell go env GOOS)
ARCH ?= $(shell go env GOARCH)

build:
	GOOS=${OS} GOARCH=${ARCH} go build -o apiserver main.go

docker-login:
	docker login ${IMAGE_REGISTRY} -u'${IMAGE_REGISTRY_USER}' -p'${IMAGE_REGISTRY_PASS}'

image: build
	docker build -t ${IMAGE_REPO_BASE}/apiserver:${VERSION} .
	docker tag ${IMAGE_REPO_BASE}/apiserver:${VERSION} ${IMAGE_REPO_BASE}/apiserver:latest
	docker push ${IMAGE_REPO_BASE}/apiserver:${VERSION}
	docker push ${IMAGE_REPO_BASE}/apiserver:latest