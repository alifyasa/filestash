# STEP2: BUILD BACKEND
FROM golang:1.24-trixie AS builder_backend
WORKDIR /home/filestash/

# 1. Install build-time system dependencies
# This layer is cached unless the package list changes.
RUN apt-get update > /dev/null && \
    apt-get install -y curl make > /dev/null 2>&1 && \
    apt-get install -y libjpeg-dev libtiff-dev libpng-dev libwebp-dev libraw-dev libheif-dev libgif-dev libvips-dev > /dev/null 2>&1

# 2. Copy Go module files
# This layer is cached if these files don't change.
COPY go.mod go.sum ./

# 3. Download Go dependencies
# This runs only if go.mod or go.sum change, caching the modules.
RUN go mod download

# 4. Copy the entire source code
# This is required for 'go generate' in the build_init step.
# This layer is invalidated on any source code change.
COPY . .

# 5. Run build_init (generates code) and build_backend (compiles)
# This will use the cached Go modules from step 3.
# It re-runs if the source code from step 4 changes.
RUN make build_init && \
    make build_backend

# 6. Prepare the config file in the dist directory
RUN mkdir -p ./dist/data/state/config/ && \
    cp config/config.json ./dist/data/state/config/config.json

# STEP3: BUILD PROD IMAGE
FROM debian:stable-slim
MAINTAINER mickael@kerjean.me
WORKDIR /app/

# 1. Install runtime dependencies and create the user
# This is done *before* copying the app.
# This layer will be cached and not re-run on every build.
RUN apt-get update > /dev/null && \
    apt-get install -y --no-install-recommends apt-utils && \
    apt-get install -y curl ffmpeg libjpeg-dev libtiff-dev libpng-dev libwebp-dev libraw-dev libheif-dev libgif-dev && \
    useradd filestash && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /tmp/*

# 2. Copy the built artifact from the builder stage
# This layer is invalidated on every new build.
COPY --from=builder_backend /home/filestash/dist/ .

# 3. Set permissions on the copied files
# This MUST run *after* the files are copied.
# This is separate from step 1 so package installation is not re-run.
RUN chown -R filestash:filestash /app/ && \
    find /app/data/ -type d -exec chmod 770 {} \; && \
    find /app/data/ -type f -exec chmod 760 {} \; && \
    chmod 730 /app/filestash

USER filestash
CMD ["/app/filestash"]
EXPOSE 8334
