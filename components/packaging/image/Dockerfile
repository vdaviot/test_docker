# Common builder
FROM golang:1.10-alpine3.7 as builder

#COPY hack/dockerfile/binaries-commits /

COPY hack/dockerfile/install/tini.installer /
COPY hack/dockerfile/install/proxy.installer /
RUN apk --update add bash btrfs-progs-dev gcc libc-dev linux-headers \
    git cmake make ca-certificates libltdl libtool libgcc && \
    grep "_COMMIT=" /*.installer  |cut -f2- -d: > /binaries-commits

# dockerd
FROM builder as dockerd-builder
WORKDIR /go/src/github.com/docker/docker
COPY . /go/src/github.com/docker/docker
ARG VERSION
ENV VERSION ${VERSION}
ARG DOCKER_GITCOMMIT
ENV DOCKER_GITCOMMIT ${DOCKER_GITCOMMIT}
# TODO The way we set the version could easily be simplified not to depend on hack/...
RUN bash ./hack/make/.go-autogen
RUN go build -o /sbin/dockerd \
    -tags 'autogen netgo static_build selinux journald' \
    -installsuffix netgo -a -buildmode=pie -ldflags '-w -extldflags "-static" ' \
    github.com/docker/docker/cmd/dockerd

# docker-proxy
# TODO if libnetwork folds into the docker tree this can be combined above
FROM builder as proxy-builder
RUN git clone https://github.com/docker/libnetwork.git /go/src/github.com/docker/libnetwork
WORKDIR /go/src/github.com/docker/libnetwork
RUN source /binaries-commits && \
    git checkout -q "$LIBNETWORK_COMMIT" && \
    go build -buildmode=pie -ldflags="$PROXY_LDFLAGS" \
        -o /sbin/docker-proxy \
        github.com/docker/libnetwork/cmd/proxy

# docker-init - TODO move this out, last time we bumped was 2016!
FROM builder as init-builder
RUN git clone https://github.com/krallin/tini.git /tini
WORKDIR /tini
RUN source /binaries-commits && \
    git checkout -q "$TINI_COMMIT" && \
    cmake . && make tini-static && \
    cp tini-static /sbin/docker-init

# runc
FROM builder as runc-builder
RUN apk --update add libseccomp-dev
RUN git clone https://github.com/opencontainers/runc.git /go/src/github.com/opencontainers/runc
WORKDIR /go/src/github.com/opencontainers/runc
RUN source /binaries-commits && \
    git checkout -q "$RUNC_COMMIT" && \
    make BUILDTAGS='seccomp apparmor' && make install

# Final docker image
FROM alpine:3.7
RUN apk --update add ca-certificates iptables net-tools libseccomp git && \
    rm -rf /var/cache/apk/*
COPY --from=dockerd-builder /sbin/dockerd /usr/local/sbin/
COPY --from=proxy-builder /sbin/docker-proxy /usr/local/sbin/
COPY --from=init-builder /sbin/docker-init /usr/local/sbin/
COPY --from=runc-builder /usr/local/sbin/runc /usr/local/sbin/
ENTRYPOINT ["/usr/local/sbin/dockerd"]
