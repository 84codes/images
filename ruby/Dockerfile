ARG RUBY_VERSION=3.4.8
FROM registry.docker.com/library/ruby:$RUBY_VERSION-slim as base

# The app lives here
WORKDIR /app

# Install base packages
RUN apt-get update -qq \
    && apt-get install --no-install-recommends -y \
    curl \
    postgresql-client \
    && rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Set production environment
ENV BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_CACHE_PATH="/usr/local/bundle/cache" \
    BUNDLE_WITHOUT="development"

# Throw-away build stage to reduce size of final image
FROM base as build

# Install packages needed to build gems
RUN apt-get update -qq \
    && apt-get install --no-install-recommends -y \
    build-essential \
    git \
    libpq-dev \
    libyaml-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists /var/cache/apt/archives

# This can be overwritten at runtime
CMD ["ruby", "-v"]
