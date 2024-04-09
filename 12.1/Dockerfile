# This Dockerfile builds Bitcoin Core and packages it into a minimal `final` image

# VERSION of Bitcoin Core to be build
#   NOTE: Unlike our other images this one is NOT prefixed with `v`,
#           as many things (like download URLs) use this form instead.
ARG VERSION=0.12.1

# CPU architecture to build binaries for
ARG ARCH

# $USER name, and data $DIR to be used in the `final` image
ARG USER=bitcoind
ARG DIR=/data

FROM debian:bookworm as download

RUN apt update
RUN apt install gpg -y

# Guix Builder Keys: https://github.com/bitcoin-core/guix.sigs/tree/main/builder-keys
# curl -s "https://api.github.com/repos/bitcoin-core/guix.sigs/contents/builder-keys" | jq -r '.[].download_url'
ENV KEYS 01ea5486de18a882d4c2684590c8019e36c2e964

RUN gpg --keyserver keyserver.ubuntu.com --recv-keys $KEYS

WORKDIR /tmp

ARG VERSION

# Download sigs
ADD https://bitcoincore.org/bin/bitcoin-core-0.12.1/SHA256SUMS.asc ./
# Download checksums

# Download source code
ADD https://bitcoincore.org/bin/bitcoin-core-0.12.1/bitcoin-0.12.1-linux64.tar.gz ./bitcoin-$VERSION.tar.gz

# Verify that hashes are signed with the previously imported key
RUN gpg --verify SHA256SUMS.asc

# Extract
RUN tar -xzf "bitcoin-$VERSION.tar.gz" && \
    rm -f "bitcoin-$VERSION.tar.gz"

FROM debian:bookworm as final

ARG VERSION=0.12.1

COPY --from=download /tmp/bitcoin-$VERSION/bin/*  /usr/local/bin/


# Prevents `VOLUME $DIR/.bitcoind/` being created as owned by `root`
RUN mkdir -p "/root/.bitcoin/"

# Expose volume containing all `bitcoind` data
VOLUME /root/.bitcoin/

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
