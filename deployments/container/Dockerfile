# Copyright (c) 2019-2022, NVIDIA CORPORATION.  All rights reserved.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG GOLANG_VERSION=1.22.6
FROM nvcr.io/nvidia/cuda:12.9.1-base-ubi9 AS build

RUN yum install -y \
    wget make git gcc \
     && \
    rm -rf /var/cache/yum/*

ARG GOLANG_VERSION=x.x.x
RUN set -eux; \
    \
    arch="$(uname -m)"; \
    case "${arch##*-}" in \
        x86_64 | amd64) ARCH='amd64' ;; \
        ppc64el | ppc64le) ARCH='ppc64le' ;; \
        aarch64) ARCH='arm64' ;; \
        *) echo "unsupported architecture" ; exit 1 ;; \
    esac; \
    wget -nv -O - https://storage.googleapis.com/golang/go${GOLANG_VERSION}.linux-${ARCH}.tar.gz \
    | tar -C /usr/local -xz

ENV GOPATH=/go
ENV PATH=$GOPATH/bin:/usr/local/go/bin:$PATH

WORKDIR /build
COPY . .

RUN mkdir /artifacts
ARG VERSION="N/A"
ARG GIT_COMMIT="unknown"
RUN make PREFIX=/artifacts cmds

# We use the ubi9-minimal image as a reference image in removing unneeded dependencies.
FROM redhat/ubi9-minimal:latest AS minimal

RUN rpm -qa --queryformat='^%{NAME}-\[0-9\].*\.%{ARCH}$\n' | sort -u > /tmp/package-names.minimal
RUN rpm -qa | sort -u > /tmp/package-list.minimal

FROM nvcr.io/nvidia/distroless/go:v3.1.10-dev

USER 0:0
SHELL ["/busybox/sh", "-c"]
RUN ln -s /busybox/sh /bin/sh

ENV NVIDIA_DISABLE_REQUIRE="true"
ENV NVIDIA_VISIBLE_DEVICES=all
ENV NVIDIA_DRIVER_CAPABILITIES=compute,utility

ARG VERSION="N/A"
ARG GIT_COMMIT="unknown"

LABEL io.k8s.display-name="NVIDIA Device Plugin"
LABEL name="NVIDIA Device Plugin"
LABEL vendor="NVIDIA"
LABEL version=${VERSION}
LABEL com.nvidia.git-commit=${GIT_COMMIT}
LABEL release="N/A"
LABEL summary="NVIDIA device plugin for Kubernetes"
LABEL description="See summary"

COPY LICENSE /licenses/

COPY --from=build /artifacts/config-manager         /usr/bin/config-manager
COPY --from=build /artifacts/gpu-feature-discovery  /usr/bin/gpu-feature-discovery
COPY --from=build /artifacts/mps-control-daemon     /usr/bin/mps-control-daemon
COPY --from=build /artifacts/nvidia-device-plugin   /usr/bin/nvidia-device-plugin

ENTRYPOINT ["nvidia-device-plugin"]
