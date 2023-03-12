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

golang:
    FROM golang:1.20.1
    SAVE ARTIFACT /usr/local/go/bin/go /go

emissary-build-container:
    FROM python:3.10-buster
    RUN export GOROOT=/usr/local/go
    RUN export GOPATH=/go
    RUN export PATH=/go/bin:$PATH
    RUN mkdir -p ${GOPATH}/src ${GOPATH}/bin /usr/local/go
    RUN apt-get update && apt-get install --yes libarchive-tools
    COPY +golang/go /usr/local/bin/go

emissary-dockerfile:
    FROM +emissary-build-container
    COPY --dir +emissary-build-context/emissary /build/
    WORKDIR /build/emissary
    WITH DOCKER
        RUN make images
    END
    SAVE IMAGE emissary.local/emissary:3.5

test:
    FROM earthly/dind:alpine
    WITH DOCKER \
        --load +emissary-dockerfile
          RUN sleep 30 &&  docker run -d -it emissary.local/emissary:3.5 /bin/bash
    END

all:
    BUILD +emissary-dockerfile
    BUILD +test
