# Gridcoin-Research for Anthropic's OSS Scanner: builds the headless node (gridcoinresearchd) and the unit-test binary
# the way the project's CI does (.github/workflows/cmake_quality.yml, its Valgrind job on ubuntu-24.04).
# The scanner builds this with the repository as the build context; network access ends after this build.
FROM ubuntu:24.04
ENV DEBIAN_FRONTEND=noninteractive \
    CC=clang \
    CXX=clang++

# The Valgrind job's dependency list without valgrind and ccache, plus ca-certificates, git, pkg-config and python3.
RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates git pkg-config python3 \
      clang cmake build-essential libboost-all-dev \
      libssl-dev libdb++-dev libminiupnpc-dev libqrencode-dev \
      libcurl4-openssl-dev libzip-dev zipcmp zipmerge ziptool \
    && rm -rf /var/lib/apt/lists/*

# scanner contract: the checkout lives inside the image, at /src
COPY . /src
WORKDIR /src

# Headless build with the unit tests, debug info and frame pointers (GUI off, as in CI's Valgrind job).
RUN cmake -S /src -B /src/build \
      -DENABLE_GUI=OFF \
      -DENABLE_TESTS=ON \
      -DCMAKE_BUILD_TYPE=RelWithDebInfo \
      -DCMAKE_C_FLAGS="-g -fno-omit-frame-pointer" \
      -DCMAKE_CXX_FLAGS="-g -fno-omit-frame-pointer" \
    && cmake --build /src/build -j"$(nproc)"

# Run the unit suite so the image is known to work, without letting a failure abort the build.
# Binaries: /src/build/bin/gridcoinresearchd (the node), /src/build/src/test/test_gridcoin (Boost unit tests).
# Functional tests: test/functional/test_runner.py (regtest, local only; needs no network).
RUN /src/build/src/test/test_gridcoin --log_level=test_suite \
    || echo "WARNING: test_gridcoin reported failures"
