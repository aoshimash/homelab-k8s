FROM python:3.12-slim

ENV DEBIAN_FRONTEND=noninteractive
ENV TZ=Asia/Tokyo

# Install system dependencies
RUN apt-get update && apt-get install -y \
    ffmpeg \
    curl \
    tzdata \
    && rm -rf /var/lib/apt/lists/*

# Install yt-dlp and yt-dlp-rajiko plugin
RUN pip install --no-cache-dir yt-dlp yt-dlp-rajiko

WORKDIR /output
