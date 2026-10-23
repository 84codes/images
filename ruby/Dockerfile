ARG RUBY_VERSION

# Which of the stages below the image is built on:
#   official  the official ruby:$RUBY_VERSION-slim image from Docker Hub (default)
#   source    Ruby built from source, for releases that
#             https://github.com/docker-library/ruby has not published yet.
#             Also needs RUBY_DOWNLOAD_SHA256, the sha256 of ruby-$RUBY_VERSION.tar.xz
#             from https://www.ruby-lang.org/en/downloads/
ARG BASE_IMAGE=official
ARG RUBY_DOWNLOAD_SHA256

FROM registry.docker.com/library/ruby:$RUBY_VERSION-slim AS official

# Copied from 4.0/slim-trixie/Dockerfile in https://github.com/docker-library/ruby
# (pull request 531, commit 83b3821), which is what ruby:X.Y.Z-slim is built from.
# Changes: the RUBY_* ENV lines come from build args instead of being hardcoded,
# --enable-zjit is only passed for Ruby 4 and later (as in upstream's 3.x
# Dockerfiles), and ENV uses key=value syntax.
FROM debian:trixie-slim AS source

RUN set -eux; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		ca-certificates \
	; \
	apt-get dist-clean

# skip installing gem documentation with `gem install`/`gem update`
RUN set -eux; \
	mkdir -p /usr/local/etc; \
	echo 'gem: --no-document' >> /usr/local/etc/gemrc

ENV LANG=C.UTF-8

ARG RUBY_VERSION
ARG RUBY_DOWNLOAD_SHA256
ENV RUBY_VERSION=$RUBY_VERSION
ENV RUBY_DOWNLOAD_SHA256=$RUBY_DOWNLOAD_SHA256

# some of ruby's build scripts are written in ruby
#   we purge system ruby later to make sure our final image uses what we just built
RUN set -eux; \
	\
	if [ -z "$RUBY_DOWNLOAD_SHA256" ]; then \
		echo >&2 'error: the RUBY_DOWNLOAD_SHA256 build arg is required when BASE_IMAGE=source'; \
		exit 1; \
	fi; \
	\
	savedAptMark="$(apt-mark showmanual)"; \
	apt-get update; \
	apt-get install -y --no-install-recommends \
		dpkg-dev \
		libgdbm-dev \
		ruby \
		autoconf \
		bzip2 \
		g++ \
		gcc \
		libbz2-dev \
		libffi-dev \
		libgdbm-compat-dev \
		libglib2.0-dev \
		libgmp-dev \
		libncurses-dev \
		libssl-dev \
		libxml2-dev \
		libxslt-dev \
		libyaml-dev \
		make \
		wget \
		xz-utils \
		zlib1g-dev \
	; \
	\
	rustArch=; \
	dpkgArch="$(dpkg --print-architecture)"; \
	case "$dpkgArch" in \
		'amd64') rustArch='x86_64-unknown-linux-gnu'; rustupUrl='https://static.rust-lang.org/rustup/archive/1.28.2/x86_64-unknown-linux-gnu/rustup-init'; rustupSha256='20a06e644b0d9bd2fbdbfd52d42540bdde820ea7df86e92e533c073da0cdd43c' ;; \
		'arm64') rustArch='aarch64-unknown-linux-gnu'; rustupUrl='https://static.rust-lang.org/rustup/archive/1.28.2/aarch64-unknown-linux-gnu/rustup-init'; rustupSha256='e3853c5a252fca15252d07cb23a1bdd9377a8c6f3efa01531109281ae47f841c' ;; \
	esac; \
	\
	if [ -n "$rustArch" ]; then \
		mkdir -p /tmp/rust; \
		\
		wget -O /tmp/rust/rustup-init "$rustupUrl"; \
		echo "$rustupSha256 */tmp/rust/rustup-init" | sha256sum --check --strict; \
		chmod +x /tmp/rust/rustup-init; \
		\
		export RUSTUP_HOME='/tmp/rust/rustup' CARGO_HOME='/tmp/rust/cargo'; \
		export PATH="$CARGO_HOME/bin:$PATH"; \
		/tmp/rust/rustup-init -y --no-modify-path --profile minimal --default-toolchain '1.91.1' --default-host "$rustArch"; \
		\
		rustc --version; \
		cargo --version; \
	fi; \
	\
	wget -O ruby.tar.xz "https://cache.ruby-lang.org/pub/ruby/${RUBY_VERSION%.*}/ruby-$RUBY_VERSION.tar.xz"; \
	echo "$RUBY_DOWNLOAD_SHA256 *ruby.tar.xz" | sha256sum --check --strict; \
	\
	mkdir -p /usr/src/ruby; \
	tar -xJf ruby.tar.xz -C /usr/src/ruby --strip-components=1; \
	rm ruby.tar.xz; \
	\
	cd /usr/src/ruby; \
	\
	autoconf; \
	gnuArch="$(dpkg-architecture --query DEB_BUILD_GNU_TYPE)"; \
# full RELRO: "-z now" disables lazy binding so the loader can map the GOT read-only at startup
# https://github.com/docker-library/ruby/pull/529
# https://salsa.debian.org/ruby-team/ruby/-/blob/8a55ab83e2c1783d80d0117fecbda533ac75f3e3/debian/rules#L63-67 (Debian's "hardening=+bindnow" produces the same flags)
	export LDFLAGS='-Wl,-z,relro -Wl,-z,now'; \
# ZJIT only exists in Ruby 4+
	case "$RUBY_VERSION" in \
		3.*) zjit= ;; \
		*) zjit='--enable-zjit' ;; \
	esac; \
	./configure \
		--build="$gnuArch" \
		--disable-install-doc \
		--enable-shared \
		${rustArch:+--enable-yjit} \
		${rustArch:+$zjit} \
	; \
	make -j "$(nproc)"; \
	make install; \
	\
	rm -rf /tmp/rust; \
	apt-mark auto '.*' > /dev/null; \
	apt-mark manual $savedAptMark > /dev/null; \
	find /usr/local -type f -executable -not \( -name '*tkinter*' \) -exec ldd '{}' ';' \
		| awk '/=>/ { so = $(NF-1); if (index(so, "/usr/local/") == 1) { next }; gsub("^/(usr/)?", "", so); printf "*%s\n", so }' \
		| sort -u \
		| xargs -r dpkg-query --search \
# https://manpages.debian.org/bookworm/dpkg/dpkg-query.1.en.html#S (we ignore diversions and it'll be really unusual for more than one package to provide any given .so file)
		| awk 'sub(":$", "", $1) { print $1 }' \
		| sort -u \
		| xargs -r apt-mark manual \
	; \
	apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
	apt-get dist-clean; \
	\
	cd /; \
	rm -r /usr/src/ruby; \
# verify we have no "ruby" packages installed
	if dpkg -l | grep -i ruby; then exit 1; fi; \
	[ "$(command -v ruby)" = '/usr/local/bin/ruby' ]; \
# rough smoke test
	ruby --version; \
	gem --version; \
	bundle --version

# don't create ".bundle" in all our apps
ENV GEM_HOME=/usr/local/bundle
ENV BUNDLE_SILENCE_ROOT_WARNING=1 \
	BUNDLE_APP_CONFIG="$GEM_HOME"
ENV PATH=$GEM_HOME/bin:$PATH
RUN set -eux; \
	mkdir "$GEM_HOME"; \
# adjust permissions of GEM_HOME for running "gem install" as an arbitrary user
	chmod 1777 "$GEM_HOME"

CMD [ "irb" ]

FROM $BASE_IMAGE

# The app lives here
WORKDIR /app

# Install base packages and packages needed to build gems
RUN apt-get update -qq \
    && apt-get install --no-install-recommends -y \
    build-essential \
    ca-certificates \
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
