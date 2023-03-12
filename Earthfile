VERSION 0.7
ARG --global IMAGE_BASE_NAME=emissary
ARG --global EMISSARY_REPO=github.com/emissary-ingress/emissary
ARG --global VERSION=v3.5
ARG --global CHART_VERSION=v8.5

emissary-build-context:
    FROM earthly/dind:alpine
    RUN apk add git
    GIT CLONE https://${EMISSARY_REPO}.git emissary
    SAVE ARTIFACT /emissary AS LOCAL ./emissary

kubectl:
    FROM bitnami/kubectl
    SAVE ARTIFACT /opt/bitnami/kubectl/bin/kubectl /kubectl

golang:
    FROM golang:1.20.1
    SAVE ARTIFACT /usr/local/go/bin/go /go
    SAVE ARTIFACT /usr/local/go/ /go-sdk/

emissary-build-container:
    FROM python:3.10-buster
    RUN apt-get update && apt-get install --yes libarchive-tools gawk
    COPY +golang/go /usr/local/bin/go
    COPY +golang/go-sdk /usr/local/

emissary-dockerfile:
    FROM +emissary-build-container
    COPY --dir +emissary-build-context/emissary /build/
    WORKDIR /build/emissary
    WITH DOCKER
        RUN make images && \
            docker image ls && \
            docker save emissary.local/emissary:latest | gzip > /tmp/emissary-docker.tar.gz
    END
    SAVE ARTIFACT /tmp/emissary-docker.tar.gz emissary-docker.tar.gz
    SAVE IMAGE emissary-build

test:
    FROM earthly/dind:alpine
    COPY +emissary-dockerfile/emissary-docker.tar.gz ./
    RUN apk add curl gzip
    RUN gunzip emissary-docker.tar.gz
    RUN curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-arm64 && \
                            chmod +x ./kind && \
                            mv ./kind /usr/local/bin/kind
    COPY +kubectl/kubectl /usr/local/bin/kubectl
    WITH DOCKER
          RUN sleep 30 &&  \
          kind create cluster && \
          kind load image-archive ./emissary-docker.tar && \
          kubectl create deployment emissary --image=emissary.local/emissary:latest
    END

all:
    BUILD +emissary-build-container
    BUILD +emissary-dockerfile
    BUILD +test
