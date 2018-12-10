# Build stage for BerkeleyDB
FROM alpine as berkeleydb

RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add autoconf \
    automake \
    build-base \
    libressl

ENV BERKELEYDB_VERSION=db-5.3.28.NC
ENV BERKELEYDB_PREFIX=/opt/${BERKELEYDB_VERSION}

RUN wget https://download.oracle.com/berkeley-db/${BERKELEYDB_VERSION}.tar.gz
RUN tar -xzf *.tar.gz
RUN sed s/__atomic_compare_exchange/__atomic_compare_exchange_db/g -i ${BERKELEYDB_VERSION}/src/dbinc/atomic.h
RUN mkdir -p ${BERKELEYDB_PREFIX}

WORKDIR /${BERKELEYDB_VERSION}/build_unix

RUN ../dist/configure --enable-cxx --disable-shared --with-pic --prefix=${BERKELEYDB_PREFIX}
RUN make -j8 && make install
RUN rm -rf ${BERKELEYDB_PREFIX}/docs

# Build stage for Bitcoin Cash
FROM alpine as bitcoin-cash

COPY --from=berkeleydb /opt /opt

RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add autoconf \
    automake \
    boost-dev \
    build-base \
    chrpath \
    file \
    gnupg \
    libevent-dev \
    libressl \
    libressl-dev \
    libtool \
    linux-headers \
    protobuf-dev \
    zeromq-dev
RUN set -ex \
  && for key in \
    629D7E5DDDA0512BD5860F2C5D7922BBD649C4A7 \
    3BB16D00D9A6D281591BDC76E4486356E7A81D2C \
    7A55A44F3A3239827C8A594E7D3958C44427674A \
  ; do \
    gpg --keyserver hkp://subset.pool.sks-keyservers.net --recv-keys ${fingerprint} \
  done

ENV BITCOIN_VERSION=0.18.5
ENV BITCOIN_PREFIX=/opt/bitcoin-abc-${BITCOIN_VERSION}
ENV BITCOIN_SHASUM="51219138820c466601564755d80726b7a601af9ddc6df7fb36c8f64fb5f6a671  bitcoin-abc-${BITCOIN_VERSION}.tar.gz"

RUN wget https://download.bitcoinabc.org/${BITCOIN_VERSION}/linux/src/bitcoin-abc-${BITCOIN_VERSION}.tar.gz
RUN echo "${BITCOIN_SHASUM}" | sha256sum -c
RUN tar -xzf *.tar.gz

WORKDIR /bitcoin-abc-${BITCOIN_VERSION}

RUN sed -i '/AC_PREREQ/a\AR_FLAGS=cr' src/univalue/configure.ac \
    && sed -i '/AX_PROG_CC_FOR_BUILD/a\AR_FLAGS=cr' src/secp256k1/configure.ac \
    && sed -i s:sys/fcntl.h:fcntl.h: src/compat.h
RUN ./autogen.sh
RUN ./configure LDFLAGS=-L`ls -d /opt/db*`/lib/ CPPFLAGS=-I`ls -d /opt/db*`/include/ \
    --prefix=${BITCOIN_PREFIX} \
    --mandir=/usr/share/man \
    --disable-tests \
    --disable-bench \
    --disable-ccache \
    --with-gui=no \
    --with-utils \
    --with-libs \
    --with-daemon
RUN make -j8 && make install
RUN strip ${BITCOIN_PREFIX}/bin/bitcoin-cli \
    && strip ${BITCOIN_PREFIX}/bin/bitcoind \
    && strip ${BITCOIN_PREFIX}/lib/libbitcoinconsensus.a \
    && strip ${BITCOIN_PREFIX}/lib/libbitcoinconsensus.so.0.0.0 \
    && rm ${BITCOIN_PREFIX}/bin/bitcoin-tx \
    && rm ${BITCOIN_PREFIX}/bin/bitcoin-seeder

# Build stage for compiled artifacts
FROM alpine

LABEL maintainer.0="João Fonseca (@joaopaulofonseca)" \
  maintainer.1="Pedro Branco (@pedrobranco)" \
  maintainer.2="Rui Marinho (@ruimarinho)"

RUN adduser -S bitcoin
RUN sed -i 's/http\:\/\/dl-cdn.alpinelinux.org/https\:\/\/alpine.global.ssl.fastly.net/g' /etc/apk/repositories
RUN apk --no-cache add \
  boost \
  boost-program_options \
  libevent \
  libressl \
  libzmq \
  su-exec

ENV BITCOIN_DATA=/home/bitcoin/.bitcoin
ENV BITCOIN_VERSION=0.18.5
ENV BITCOIN_PREFIX=/opt/bitcoin-abc-${BITCOIN_VERSION}
ENV PATH=${BITCOIN_PREFIX}/bin:$PATH

COPY --from=bitcoin-cash /opt /opt
COPY docker-entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh

VOLUME ["/home/bitcoin/.bitcoin"]

EXPOSE 8332 8333 18332 18333 18444

ENTRYPOINT ["/entrypoint.sh"]

CMD ["bitcoind"]