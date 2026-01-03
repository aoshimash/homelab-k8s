FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo

RUN apt-get update && apt-get install -y \
    curl \
    wget \
    unzip \
    ffmpeg \
    rtmpdump \
    tzdata \
    jq \
    libxml2-utils \
    && rm -rf /var/lib/apt/lists/*

# Install radigo
ARG RADIGO_VERSION=v0.12.0
RUN curl -L -o /tmp/radigo.zip https://github.com/yyoshiki41/radigo/releases/download/${RADIGO_VERSION}/radigo_${RADIGO_VERSION}_linux_amd64.zip \
    && unzip -d /usr/local/bin /tmp/radigo.zip \
    && chmod +x /usr/local/bin/radigo \
    && rm /tmp/radigo.zip

WORKDIR /output
