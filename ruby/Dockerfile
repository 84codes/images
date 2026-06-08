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
    locales \
    pkg-config \
    postgresql-client \
    && sed -i '/en_US.UTF-8/s/^# //' /etc/locale.gen \
    && locale-gen \
    && rm -rf /var/lib/apt/lists /var/cache/apt/archives

# Non-root user for downstream runtime stages
RUN useradd --system --uid 1000 app && chown app:app /app

# Set production environment
ENV BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_CACHE_PATH="/usr/local/bundle/cache" \
    BUNDLE_WITHOUT="development" \
    LANG="C.UTF-8"

# This can be overwritten at runtime
CMD ["ruby", "-v"]
