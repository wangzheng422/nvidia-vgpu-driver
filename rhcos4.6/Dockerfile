FROM ubuntu:20.04 as build

RUN apt-get update && apt-get install -y --no-install-recommends \
        g++ \
        ca-certificates \
        wget \
        git && \
    rm -rf /var/lib/apt/lists/*

ENV GOLANG_VERSION 1.15.5
RUN wget -nv -O - https://storage.googleapis.com/golang/go${GOLANG_VERSION}.linux-amd64.tar.gz \
    | tar -C /usr/local -xz
ENV PATH /usr/local/go/bin:$PATH

WORKDIR /work

RUN git clone https://gitlab.com/nvidia/container-images/driver && \
    cd driver/vgpu/src && \
    go build -o vgpu-util && \
    mv vgpu-util /work

FROM registry.access.redhat.com/ubi8:8.2

#ARG BASE_URL=http://us.download.nvidia.com/XFree86/Linux-x86_64
ARG BASE_URL=https://us.download.nvidia.com/tesla
ARG DRIVER_VERSION=450.80.02
ENV DRIVER_VERSION=$DRIVER_VERSION
ARG DRIVER_ARCH=x86_64
ENV DRIVER_ARCH=$DRIVER_ARCH

# Arg to indicate if driver type is either of passthrough/baremetal or vgpu
ARG DRIVER_TYPE=passthrough
ENV DRIVER_TYPE=$DRIVER_TYPE
ARG VGPU_LICENSE_SERVER_TYPE=FNE
ENV VGPU_LICENSE_SERVER_TYPE=$VGPU_LICENSE_SERVER_TYPE
# Enable vGPU version compability check by default
ARG DISABLE_VGPU_VERSION_CHECK=false
ENV DISABLE_VGPU_VERSION_CHECK=$DISABLE_VGPU_VERSION_CHECK

ADD install.sh /tmp/

RUN sh /tmp/install.sh depinstall && \
    curl -fsSL -o /usr/local/bin/donkey https://github.com/3XX0/donkey/releases/download/v1.1.0/donkey && \
    curl -fsSL -o /usr/local/bin/extract-vmlinux https://raw.githubusercontent.com/torvalds/linux/master/scripts/extract-vmlinux && \
    chmod +x /usr/local/bin/donkey /usr/local/bin/extract-vmlinux && \
    ln -s /sbin/ldconfig /sbin/ldconfig.real

ADD drivers drivers/

# Fetch the installer automatically for passthrough/baremetal types
RUN if [ "$DRIVER_TYPE" != "vgpu" ]; then cd drivers && \
    curl -fSsl -O $BASE_URL/$DRIVER_VERSION/NVIDIA-Linux-$DRIVER_ARCH-$DRIVER_VERSION.run && \
    chmod +x  NVIDIA-Linux-$DRIVER_ARCH-$DRIVER_VERSION.run; fi

COPY nvidia-driver /usr/local/bin

COPY --from=build /work/vgpu-util /usr/local/bin

WORKDIR /drivers

ARG PUBLIC_KEY=empty
COPY ${PUBLIC_KEY} kernel/pubkey.x509

ARG PRIVATE_KEY
ARG KERNEL_VERSION=latest

LABEL io.k8s.display-name="NVIDIA Driver Container"
LABEL name="NVIDIA Driver Container"
LABEL vendor="NVIDIA"
LABEL version="${DRIVER_VERSION}"
LABEL release="N/A"
LABEL summary="Provision the NVIDIA driver through containers"
LABEL description="See summary"

COPY LICENSE /licenses/LICENSE
COPY DRIVER-LICENSE /licenses/DRIVER-LICENSE

ENTRYPOINT ["nvidia-driver", "init"]
