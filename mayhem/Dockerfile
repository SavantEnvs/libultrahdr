# syntax=docker/dockerfile:1.7
#
# Commit-image Dockerfile (SMALL form) — builds ON TOP of the org C/C++ base image
# (ghcr.io/mayhemheroes/base). The base provides debian + clang/llvm + libFuzzer/afl++ + cmake/ninja
# + the identities + the build-contract ENV; this only adds the per-commit libultrahdr build.
#
# Build (from the repo root):
#   DOCKER_BUILDKIT=1 docker build -f mayhem/Dockerfile \
#       --build-arg COMMIT=$(git rev-parse HEAD) \
#       [--build-arg SOURCE_DATE_EPOCH=$(date +%s)] \
#       -t libultrahdr-commit:test .

FROM ghcr.io/savantenvs/base:20260621

ARG COMMIT
ARG REPO_URL
# SOURCE_DATE_EPOCH stays an ARG (available to build.sh); never bake it as ENV — an empty value
# breaks clang. build.sh unsets it when empty.
ARG SOURCE_DATE_EPOCH

# Sanitizer override: the base exports SANITIZER_FLAGS (ASan+UBSan, halting) as the default.
# Declare it WITHOUT a default so an unset arg lets the base ENV flow through; --build-arg overrides.
ARG SANITIZER_FLAGS
ARG MAYHEM_JOBS=

# libultrahdr builds with CMake (shipped by the base). Its sole hard dependency is libjpeg-turbo,
# which the build compiles FROM SOURCE (UHDR_BUILD_FUZZERS force-enables UHDR_BUILD_DEPS so the dep
# is instrumented too) — that needs nasm for libjpeg-turbo's x86_64 SIMD. Install nasm as root if
# the base doesn't already ship it, then drop back to the mayhem user.
USER root
RUN if ! command -v nasm >/dev/null 2>&1; then \
      [ -n "$SOURCE_DATE_EPOCH" ] && repro-sources-list.sh || true; \
      apt-get update && apt-get install -yq --no-install-recommends nasm \
      && rm -rf /var/lib/apt/lists/*; \
    fi
# Offline dependency mirror (SPEC §6.5 / §6.2 item 16). upstream's CMake ExternalProject clones
# libjpeg-turbo from GitHub into the gitignored third_party/turbojpeg; build.sh is re-run offline
# on a `git clean -ffdX` tree, so it clones from this in-image mirror instead (see build.sh).
# The tag's commit is pinned.
RUN git clone --quiet --bare --branch 3.1.0 https://github.com/libjpeg-turbo/libjpeg-turbo.git \
      /opt/mayhem/deps/libjpeg-turbo.git \
    && [ "$(git --git-dir=/opt/mayhem/deps/libjpeg-turbo.git rev-parse '3.1.0^{commit}')" = 20ade4dea9589515a69793e447a6c6220b464535 ] \
    && chmod -R a+rX /opt/mayhem/deps \
    && git config --system --add safe.directory /opt/mayhem/deps/libjpeg-turbo.git
USER mayhem

# Source enters from the build CONTEXT (the CI checkout): the FULL repo (incl. .git history)
# is copied to /mayhem as `mayhem`; build.sh builds in place.
WORKDIR /mayhem
COPY --chown=2000:2000 . /mayhem

RUN bash mayhem/build.sh

# Run libultrahdr's self-contained round-trip golden test at build time (a failing test fails the build).
RUN if [ -f mayhem/test.sh ]; then bash mayhem/test.sh; else echo "no mayhem/test.sh"; fi

LABEL org.opencontainers.image.source="${REPO_URL}" \
      org.opencontainers.image.revision="${COMMIT}" \
      rlenv.tier="commit" \
      rlenv.source_date_epoch="${SOURCE_DATE_EPOCH}" \
      rlenv.user="mayhem"

USER mayhem
WORKDIR /mayhem
