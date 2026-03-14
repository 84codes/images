ARG RUBY_VERSION
FROM registry.docker.com/library/ruby:$RUBY_VERSION-slim

# The app lives here
WORKDIR /app

# Install base packages and packages needed to build gems
RUN apt-get update -qq \
    && apt-get install --no-install-recommends -y \
    build-essential \
    curl \
    git \
    libpq-dev \
    libyaml-dev \
    pkg-config \
    postgresql-client \
    && rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Non-root user for downstream runtime stages
RUN useradd --system --uid 1000 app && chown app:app /app

# Set production environment
ENV BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_CACHE_PATH="/usr/local/bundle/cache" \
    BUNDLE_WITHOUT="development"

# This can be overwritten at runtime
CMD ["ruby", "-v"]
