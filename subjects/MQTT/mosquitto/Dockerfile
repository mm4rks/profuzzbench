FROM ubuntu:20.04

# Install common dependencies
ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get -y update && \
    apt-get -y install sudo \ 
    apt-utils \
    build-essential \
    openssl \
    clang \
    graphviz-dev \
    git \
    autoconf \
    libgnutls28-dev \
    libssl-dev \
    llvm \
    python3-pip \
    nano \
    net-tools \
    vim \
    gdb \
    netcat \
    strace \
    cmake \
    wget

# Add a new user ubuntu, pass: ubuntu
RUN groupadd ubuntu && \
    useradd -rm -d /home/ubuntu -s /bin/bash -g ubuntu -G sudo -u 1000 ubuntu -p "$(openssl passwd -1 ubuntu)"

RUN chmod 777 /tmp

RUN pip install gcovr==4.2

# Use ubuntu as default username
USER ubuntu
WORKDIR /home/ubuntu

# Import environment variable to pass as parameter to make (e.g., to make parallel builds with -j)
ARG MAKE_OPT

# Set up fuzzers
# TODO after PR change this!
ARG DUMMY=unknown
RUN git clone https://github.com/mm4rks/aflnet.git && \
    cd aflnet && \
    make clean all $MAKE_OPT && \
    cd llvm_mode && make $MAKE_OPT

RUN git clone https://github.com/profuzzbench/aflnwe.git && \
    cd aflnwe && \
    make clean all $MAKE_OPT && \
    cd llvm_mode && make $MAKE_OPT

# Set up environment variables for AFLNet
ENV WORKDIR="/home/ubuntu/experiments"
ENV AFLNET="/home/ubuntu/aflnet"
ENV PATH="${PATH}:${AFLNET}:/home/ubuntu/.local/bin:${WORKDIR}"
ENV AFL_PATH="${AFLNET}"
ENV AFL_I_DONT_CARE_ABOUT_MISSING_CRASHES=1 \
    AFL_SKIP_CPUFREQ=1 \
    AFL_NO_AFFINITY=1


RUN mkdir $WORKDIR

COPY --chown=ubuntu:ubuntu fuzzing.patch ${WORKDIR}/fuzzing.patch
COPY --chown=ubuntu:ubuntu gcov.patch ${WORKDIR}/gcov.patch

USER root
RUN apt -y install automake
# USER ubuntu

# Set up environment variables for ASAN
ENV ASAN_OPTIONS='abort_on_error=1:symbolize=0:detect_leaks=0:detect_stack_use_after_return=1:detect_container_overflow=0:poison_array_cookie=0:malloc_fill_byte=0:max_malloc_fill_size=16777216'


ENV LWS_VERSION=4.2.1 \
    CJSON_VERSION=1.7.14 \
    MOSQUITTO_VERSION=2.0.14

# build deps
RUN wget https://github.com/warmcat/libwebsockets/archive/v${LWS_VERSION}.tar.gz -O /tmp/lws.tar.gz && \
    mkdir -p /build/lws && \
    tar --strip=1 -xf /tmp/lws.tar.gz -C /build/lws && \
    rm /tmp/lws.tar.gz && \
    cd /build/lws && \
    cmake . \
        -DCMAKE_BUILD_TYPE=MinSizeRel \
        -DCMAKE_INSTALL_PREFIX=/usr \
        -DDISABLE_WERROR=ON \
        -DLWS_IPV6=ON \
        -DLWS_WITHOUT_BUILTIN_GETIFADDRS=ON \
        -DLWS_WITHOUT_CLIENT=ON \
        -DLWS_WITHOUT_EXTENSIONS=ON \
        -DLWS_WITHOUT_TESTAPPS=ON \
        -DLWS_WITH_EXTERNAL_POLL=ON \
        -DLWS_WITH_HTTP2=OFF \
        -DLWS_WITH_SHARED=OFF \
        -DLWS_WITH_ZIP_FOPS=OFF \
        -DLWS_WITH_ZLIB=OFF && \
    make -j "$(nproc)" && \
    rm -rf /root/.cmake && \
    wget https://github.com/DaveGamble/cJSON/archive/v${CJSON_VERSION}.tar.gz -O /tmp/cjson.tar.gz && \
    mkdir -p /build/cjson && \
    tar --strip=1 -xf /tmp/cjson.tar.gz -C /build/cjson && \
    rm /tmp/cjson.tar.gz && \
    cd /build/cjson && \
    make -j "$(nproc)" libcjson.a

# Set up mosquitto for fuzzing
# TODO? git checkout commit instead of download release?
# TODO patch
RUN wget https://github.com/eclipse/mosquitto/archive/refs/tags/v${MOSQUITTO_VERSION}.tar.gz -O /tmp/mosq.tar.gz && \
    mkdir -p ${WORKDIR}/mosquitto && \
    tar --strip=1 -xf /tmp/mosq.tar.gz -C ${WORKDIR}/mosquitto && \
    make -C ${WORKDIR}/mosquitto -j "$(nproc)" \
        CC="afl-clang-fast" \
        CXX="afl-clang-fast++" \
        CFLAGS="-fsanitize=address -g -O0 -I/build/lws/include -I/build" \
        CXXFLAGS="-fsanitize=address -g -O0 -I/build/lws/include -I/build" \
        LDFLAGS="-L/build/lws/lib -L/build/cjson -fsanitize=address" \
        WITH_ADNS=no \
        WITH_CJSON=yes \
        WITH_DOCS=no \
        WITH_SHARED_LIBRARIES=yes \
        WITH_SRV=no \
        WITH_STRIP=yes \
        WITH_WEBSOCKETS=yes \
        binary

# Set up mosquitto for coverage analysis
RUN mkdir -p ${WORKDIR}/mosquitto-gcov && \
    tar --strip=1 -xf /tmp/mosq.tar.gz -C ${WORKDIR}/mosquitto-gcov && \
    make -C ${WORKDIR}/mosquitto-gcov -j "$(nproc)" \
        CFLAGS="-fprofile-arcs -ftest-coverage -I/build/lws/include -I/build" \
        CPPFLAGS="-fprofile-arcs -ftest-coverage -I/build/lws/include -I/build" \
        CXXFLAGS="-fprofile-arcs -ftest-coverage -I/build/lws/include -I/build" \
        LDFLAGS="-L/build/lws/lib -L/build/cjson -fprofile-arcs -ftest-coverage" \
        WITH_ADNS=no \
        WITH_CJSON=yes \
        WITH_DOCS=no \
        WITH_SHARED_LIBRARIES=yes \
        WITH_SRV=no \
        WITH_STRIP=yes \
        WITH_WEBSOCKETS=yes \
        binary

COPY --chown=ubuntu:ubuntu in-mqtt ${WORKDIR}/in-mqtt
COPY --chown=ubuntu:ubuntu mqtt.dict ${WORKDIR}/mqtt.dict
COPY --chown=ubuntu:ubuntu run.sh ${WORKDIR}/run
COPY --chown=ubuntu:ubuntu clean.sh ${WORKDIR}/clean
COPY --chown=ubuntu:ubuntu cov_script.sh ${WORKDIR}/cov_script

# Switch default user to root
USER root

# add user mosquitto to root group (needed for mosquitto dropping privileges and writing gcov files)
RUN useradd mosquitto && usermod -a -G root mosquitto
WORKDIR /home/ubuntu

# RUN useradd -rm -d /home/fuzzing -s /bin/bash -g ubuntu -G sudo -u 1001 fuzzing -p "$(openssl passwd -1 fuzzing)"
