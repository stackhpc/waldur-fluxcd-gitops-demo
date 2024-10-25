#####
# This Dockerfile is based on a combination of docker-stacks Pytorch Dockerfile here:
#
# and Intel's Pytorch Dockerfile here:
# https://raw.githubusercontent.com/intel/intel-extension-for-pytorch/v2.1.40%2Bxpu/docker/Dockerfile.prebuilt
#####

ARG REGISTRY=quay.io
ARG OWNER=jupyter
ARG BASE_CONTAINER=$REGISTRY/$OWNER/scipy-notebook
FROM $BASE_CONTAINER

#####
# Intel GPU components
#####

USER root

ENV LANG=C.UTF-8

ARG DEBIAN_FRONTEND=noninteractive

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        apt-utils \
        build-essential \
        ca-certificates \
        clinfo \
        curl \
        git \
        gnupg2 \
        gpg-agent \
        rsync \
        sudo \
        unzip \
        wget && \
    apt-get clean && \
    rm -rf  /var/lib/apt/lists/*

RUN wget -qO - https://repositories.intel.com/gpu/intel-graphics.key | \
    gpg --dearmor --yes --output /usr/share/keyrings/intel-graphics.gpg
RUN echo "deb [arch=amd64 signed-by=/usr/share/keyrings/intel-graphics.gpg] https://repositories.intel.com/gpu/ubuntu jammy unified" | \
    tee /etc/apt/sources.list.d/intel-gpu-jammy.list &&\
    apt update

ARG ICD_VER=24.22.29735.27-914~22.04
ARG LEVEL_ZERO_GPU_VER=1.3.29735.27-914~22.04
ARG LEVEL_ZERO_VER=1.17.6-914~22.04
ARG LEVEL_ZERO_DEV_VER=1.17.6-914~22.04

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
    intel-opencl-icd=${ICD_VER} \
    intel-level-zero-gpu=${LEVEL_ZERO_GPU_VER} \
    libze1=${LEVEL_ZERO_VER} \
    libze-dev=${LEVEL_ZERO_DEV_VER} && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

RUN wget -O- https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB \
   | gpg --dearmor | tee /usr/share/keyrings/oneapi-archive-keyring.gpg > /dev/null && \
   echo "deb [signed-by=/usr/share/keyrings/oneapi-archive-keyring.gpg] https://apt.repos.intel.com/oneapi all main" \
   | tee /etc/apt/sources.list.d/oneAPI.list

ARG DPCPP_VER=2024.2.1-1079
ARG MKL_VER=2024.2.1-103
ARG CCL_VER=2021.13.1-31

RUN apt-get update && \
    apt-get install -y --no-install-recommends \
        intel-oneapi-runtime-dpcpp-cpp=${DPCPP_VER} \
        intel-oneapi-runtime-mkl=${MKL_VER} \
        intel-oneapi-runtime-ccl=${CCL_VER} && \
    apt-get clean && \
    rm -rf  /var/lib/apt/lists/*

# Add a group matching the Ubuntu host.
# Kubespawner profile must be configured
# with matching supplemental_gids config.
# TODO: Can we make this OS agnostic somehow?
RUN groupadd -g 110 render


#####
# User-level python components
#####

USER ${NB_USER}

# Fix: https://github.com/hadolint/hadolint/wiki/DL4006
# Fix: https://github.com/koalaman/shellcheck/wiki/SC3014
SHELL ["/bin/bash", "-o", "pipefail", "-c"]


ARG TORCH_VERSION=2.1.0.post3+cxx11.abi
ARG TORCHVISION_VERSION=0.16.0.post3+cxx11.abi
ARG TORCHAUDIO_VERSION=2.1.0.post3+cxx11.abi
ARG IPEX_VERSION=2.1.40+xpu
ARG ONECCL_BIND_PT_VERSION=2.1.400+xpu
ARG TORCH_WHL_URL=https://pytorch-extension.intel.com/release-whl/stable/xpu/us/

RUN python -m pip install setuptools==69.5.1 numpy==1.26.4

RUN python -m pip install --no-cache-dir --extra-index-url ${TORCH_WHL_URL} \
    torch==${TORCH_VERSION} \
    intel_extension_for_pytorch==${IPEX_VERSION} \
    torchvision==${TORCHVISION_VERSION} \
    torchaudio==${TORCHAUDIO_VERSION} \
    oneccl_bind_pt==${ONECCL_BIND_PT_VERSION}



