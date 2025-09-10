# This Dockerfile builds Bitcoin Knots and packages it into a minimal `final` image

# VERSION of Bitcoin Knots to be build
#   NOTE: Unlike our other images this one is NOT prefixed with `v`,
#           as many things (like download URLs) use this form instead.
ARG VERSION=29.1.knots20250903

# CPU architecture to build binaries for
ARG ARCH

# Define default versions so that they don't have to be repeated throughout the file
ARG VER_ALPINE=3.22.1

# $USER name, and data $DIR to be used in the `final` image
ARG USER=bitcoind
ARG DIR=/data

FROM alpine:${VER_ALPINE} AS preparer-git

ARG VERSION

RUN apk add --no-cache git gpg gnupg curl jq

# Fetch the source code at a specific TAG
RUN git clone  https://github.com/retropex/bitcoin.git  "/bitcoin-$VERSION/"

# Verify tag, and copy source code to predetermined location on success
RUN cd "/bitcoin-$VERSION/" && git checkout b0823e612e64a83de3145bf3ff5e4543af39a5f0 && gpg --keyserver keyserver.ubuntu.com --recv-keys DAED928C727D3E613EC46635F5073C4F4882FFFC && \ 
    git verify-commit b0823e612e64a83de3145bf3ff5e4543af39a5f0

#
## Alias to go around `COPY` not accepting ARGs in value passed to `--from=`
#
FROM preparer-git AS preparer



#
## `builder` builds Bitcoin Knots regardless on how the source, and BDB code were obtained.
#
# NOTE: this stage is emulated using QEMU
# NOTE: `${ARCH:+${ARCH}/}` - if ARCH is set, append `/` to it, leave it empty otherwise
FROM ${ARCH:+${ARCH}/}alpine:${VER_ALPINE} AS builder

ARG TARGETARCH
ARG VERSION
ARG SOURCE

RUN apk --no-cache add \
        cmake \
        automake \
        boost-dev \
        build-base \
        clang \
        chrpath \
        file \
        gnupg \
        libevent-dev \
        libressl \
        libtool \
        linux-headers \
        sqlite-dev \
        zeromq-dev \
        bash \
        curl


# Change to the extracted directory
WORKDIR /bitcoin-$VERSION/

# Copy bitcoin source (downloaded & verified in previous stages)
COPY  --from=preparer /bitcoin-$VERSION/  ./

ENV BITCOIN_PREFIX /opt/bitcoin-$VERSION

RUN make -C depends NO_BOOST=1 NO_LIBEVENT=1 NO_QT=1 NO_SQLITE=1 NO_UPNP=1 NO_ZMQ=1 NO_USDT=1

RUN if [ "$TARGETARCH" = "arm64" ]; then \
        export BDB_PREFIX="/bitcoin-$VERSION/depends/aarch64-unknown-linux-musl"; \
    elif [ "$TARGETARCH" = "amd64" ]; then \
        export BDB_PREFIX="/bitcoin-$VERSION/depends/x86_64-pc-linux-musl"; \
    fi && \
    echo "export BDB_PREFIX=$BDB_PREFIX" > /tmp/bdb_prefix.sh

RUN . /tmp/bdb_prefix.sh && \
    cmake -B build \
  -DBerkeleyDB_INCLUDE_DIR:PATH="${BDB_PREFIX}/include" -DWITH_BDB=ON \
  -DCMAKE_CXX_FLAGS="-O1" \
  -DCMAKE_CXX="clang++ CC=clang" \
  -DCMAKE_INSTALL_PREFIX=${BITCOIN_PREFIX} \
  -DINSTALL_MAN=OFF \
  -DBUILD_TESTS=OFF \
  -DBUILD_BENCH=OFF \
  -DWITH_CCACHE=OFF \
  -DBUILD_GUI=OFF \
  #--with-utils \
  -DBUILD_CLI=ON \
  -DBUILD_BITCOINCONSENSUS_LIB=ON \
  -DWITH_SQLITE=ON \
  -DBUILD_DAEMON=ON \
  -DENABLE_HARDENING=ON \
  -DREDUCE_EXPORTS=ON \
  -DWITH_ZMQ=ON

RUN cmake --build build -j$(( $(nproc) + 1 ))
RUN cmake --install build

# List installed binaries pre-strip & strip them
RUN ls -lh "$BITCOIN_PREFIX/bin/"
RUN strip -v "$BITCOIN_PREFIX/bin/bitcoin"*

# List installed binaries post-strip & print their checksums
RUN ls -lh "$BITCOIN_PREFIX/bin/"
RUN sha256sum "$BITCOIN_PREFIX/bin/bitcoin"*



#
## `final` aggregates build results from previous stages into a necessary minimum
#       ready to be used, and published to Docker Hub.
#
# NOTE: this stage is emulated using QEMU
# NOTE: `${ARCH:+${ARCH}/}` - if ARCH is set, append `/` to it, leave it empty otherwise
FROM ${ARCH:+${ARCH}/}alpine:${VER_ALPINE} AS final

ARG VERSION
ARG USER
ARG DIR

RUN apk add --no-cache \
        libevent \
        libsodium \
        libstdc++ \
        libzmq \
        sqlite-libs \
        curl

COPY  --from=builder /opt/bitcoin-$VERSION/bin/bitcoin*  /usr/local/bin/

# NOTE: Default GID == UID == 1000
RUN adduser --disabled-password \
            --home "$DIR/" \
            --gecos "" \
            "$USER"

USER $USER

# Prevents `VOLUME $DIR/.bitcoind/` being created as owned by `root`
RUN mkdir -p "$DIR/.bitcoin/"

# Expose volume containing all `bitcoind` data
VOLUME $DIR/.bitcoin/

# REST interface
EXPOSE 8080

# P2P network (mainnet, testnet & regnet respectively)
EXPOSE 8333 18333 18444

# RPC interface (mainnet, testnet & regnet respectively)
EXPOSE 8332 18332 18443

# ZMQ ports (for transactions & blocks respectively)
EXPOSE 28332 28333

ENTRYPOINT ["bitcoind"]

CMD ["-zmqpubrawblock=tcp://0.0.0.0:28332", "-zmqpubrawtx=tcp://0.0.0.0:28333"]
