VERSION 0.7
ARG --global IMAGE_BASE_NAME=emissary
ARG --global EMISSARY_REPO=github.com/emissary-ingress/emissary
ARG --global VERSION=v3.5.1
ARG --global CHART_VERSION=v8.5.1

emissary-build-context:
    FROM earthly/dind:alpine
    RUN apk add git
    GIT CLONE https://${EMISSARY_REPO}.git emissary
    SAVE ARTIFACT /emissary AS LOCAL ./emissary

kubectl:
    FROM bitnami/kubectl
    SAVE ARTIFACT /opt/bitnami/kubectl/bin/kubectl /kubectl

istioctl:
    FROM istio/istioctl:1.17.1
    SAVE ARTIFACT /usr/local/bin/istioctl /istioctl

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
            docker save emissary.local/emissary:latest -o /tmp/emissary-docker.tar
    END
    SAVE ARTIFACT /tmp/emissary-docker.tar AS LOCAL emissary-docker.tar
    SAVE IMAGE emissary-build

test:
    FROM earthly/dind:alpine
    COPY +emissary-dockerfile/emissary-docker.tar ./
    COPY --dir yaml ./
    COPY Taskfile.yml ./
    RUN apk add curl bash go-task-task
    RUN curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.17.0/kind-linux-arm64 && \
                            chmod +x ./kind && \
                            mv ./kind /usr/local/bin/kind
    COPY +kubectl/kubectl /usr/local/bin/kubectl
    COPY +istioctl/istioctl /usr/local/bin/istioctl
    RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    RUN helm repo add datawire https://app.getambassador.io && helm repo update
    WITH DOCKER
          RUN sleep 5 &&  \
          task boostrap-cluster
    END

all:
    BUILD +emissary-build-container
    BUILD +emissary-dockerfile
    BUILD +test
