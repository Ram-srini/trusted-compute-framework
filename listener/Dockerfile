# Copyright 2020 Intel Corporation
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
# ------------------------------------------------------------------------------

# -------------=== listener build ===-------------

FROM ubuntu:bionic as base_image

# Ignore timezone prompt in apt
ENV DEBIAN_FRONTEND=noninteractive

# Add necessary packages
RUN apt-get update \
 && apt-get install -y -q \
    ca-certificates \
    python3-toml \
    python3-requests \
    python3-colorlog \
    python3-twisted \
 && apt-get clean

# Make Python3 default
RUN ln -sf /usr/bin/python3 /usr/bin/python


# -------------=== python build ===-------------

#Build python intermediate docker image
FROM ubuntu:bionic as python_image


# Add necessary packages
RUN apt-get update \
 && apt-get install -y -q \
    ca-certificates \
    pkg-config \
    python3-pip \
    python3-dev \
    make \
 && apt-get clean

# Install setuptools packages using pip because
# these are not available in apt repository.
RUN pip3 install setuptools

# -------------=== common/python build ===-------------

#Build common/python intermediate docker image
FROM python_image as common_python_image

COPY VERSION /project/avalon/
COPY ./bin /project/avalon/bin

ENV TCF_HOME=/project/avalon

COPY ./common/python /project/avalon/common/python

WORKDIR /project/avalon/common/python

RUN echo "Building Avalon Common Python\n" \
 && make

# -------------=== common/crypto_utils build ===-------------

#Build common/crypto_utils intermediate docker image
FROM python_image as common_crypto_utils_image

COPY VERSION /project/avalon/
COPY ./bin /project/avalon/bin

ENV TCF_HOME=/project/avalon

COPY ./common/crypto_utils /project/avalon/common/crypto_utils

WORKDIR /project/avalon/common/crypto_utils

RUN echo "Building Avalon Common Crypto Python\n" \
 && make


# -------------=== Avalon SDK build ===-------------

#Build Avalon SDK intermediate docker image
FROM python_image as avalon_sdk_image

COPY VERSION /project/avalon/
COPY ./bin /project/avalon/bin
COPY ./sdk /project/avalon/sdk

ENV TCF_HOME=/project/avalon

WORKDIR /project/avalon/sdk

#install Avalon SDK  package.
RUN echo "Building Avalon SDK\n" \
 && make


#Build Avalon Listener intermediate docker image
FROM python_image as build_image

ENV TCF_HOME=/project/avalon

COPY VERSION /project/avalon/
COPY ./bin /project/avalon/bin
COPY ./config/tcs_config.toml /project/avalon/config/
COPY ./listener /project/avalon/listener


# Build Avalon Listener module next.

WORKDIR /project/avalon/listener/

RUN echo "Building avalon listener\n" \
  && make

# Build final image and install modules
FROM base_image as final_image

ENV TCF_HOME=/project/avalon

WORKDIR /project/avalon/listener

COPY ./listener/listener_config.toml /project/avalon/listener/listener_config.toml
COPY ./config/tcs_config.toml /project/avalon/config/
# Copy Python build artifacts
COPY --from=common_python_image /project/avalon/common/python/dist/*.whl dist/
COPY --from=common_crypto_utils_image /project/avalon/common/crypto_utils/dist/*.whl dist/
COPY --from=avalon_sdk_image /project/avalon/sdk/dist/*.whl dist/
COPY --from=build_image /project/avalon/listener/dist/*.whl dist/

# Installing wheel file requires python3-pip package.
# But python3-pip package will increase size of final docker image.
# So remove python3-pip package and dependencies after installing wheel file.
RUN apt-get update \
 && apt-get install -y -q python3-pip \
 && pip3 install json-rpc pyzmq \
 && pip3 install jsonschema \
 && echo "Install Common Python, Crypto, Listener and SDK packages\n" \
 && pip3 install dist/*.whl coverage \
 && pip3 install pycryptodomex ecdsa \
 && echo "Remove unused packages from image\n" \
 && apt-get autoremove --purge -y -q python3-pip \
 && apt-get clean \
 && rm -rf /var/lib/apt/lists/*
