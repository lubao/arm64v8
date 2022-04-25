FROM arm64v8/ubuntu:20.04 as base
RUN DEBIAN_FRONTEND=noninteractive apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y \
    net-tools vim\
    build-essential python3\
    wget \
    pkg-config \
    libmpfr-dev libmpfr6 libmpc-dev libgmp-dev libisl-dev libisl22 && \
    apt-get clean

COPY libquadmath.patch /opt/libquadmath.patch
COPY sample.cpp /opt
FROM base AS build-gcc-libquadmath
WORKDIR /opt
RUN cd /opt && \
    wget http://ftp.gnu.org/gnu/gcc/gcc-9.4.0/gcc-9.4.0.tar.gz && \
    tar xf gcc-9.4.0.tar.gz && \
    mv /opt/libquadmath.patch gcc-9.4.0/. && \
    cd gcc-9.4.0 && \
    patch -p1 -i libquadmath.patch && \
    ./configure --enable-libquadmath --disable-libada --disable-libssp --enable-languages=c,c++ && \
    make -j 2>&1 | tee /tmp/log && \
    make install && \
    echo '/usr/local/lib64' > /etc/ld.so.conf.d/libquadmath.conf && \
    ldconfig

FROM build-gcc-libquadmath as libquadmath
RUN gcc -o /opt/sample /opt/sample.cpp -lquadmath && \
    /opt/sample

RUN rm -rf /opt/gcc*