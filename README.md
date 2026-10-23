# images

Trigger build by hand

    gh workflow --repo 84codes/images run Ruby

## Ruby versions without an official image

`ruby/Dockerfile` normally builds on the official `ruby:X.Y.Z-slim` image. When
a Ruby release is out but https://github.com/docker-library/ruby has not
published an image for it yet, add it to the `include` list in
`.github/workflows/ruby.yml` with the sha256 of `ruby-X.Y.Z.tar.xz` from
https://www.ruby-lang.org/en/downloads/ and the image is built from source
instead (a copy of upstream's `slim-trixie` Dockerfile). Once the official
image exists, move the version to the plain list.
