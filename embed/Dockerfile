FROM scratch
MAINTAINER Andrew Bailey, andbaile@ucsc.edu

# https://github.com/tianon/docker-brew-ubuntu-core/tree/fc9c4ef6e3d4891577936f0b103331e79e1e8281/bionic
ADD bionic/ubuntu-bionic-core-cloudimg-amd64-root.tar.gz /

# a few minor docker-specific tweaks
# see https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap
RUN set -xe \
	\
# https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap#L40-L48
	&& echo '#!/bin/sh' > /usr/sbin/policy-rc.d \
	&& echo 'exit 101' >> /usr/sbin/policy-rc.d \
	&& chmod +x /usr/sbin/policy-rc.d \
	\
# https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap#L54-L56
	&& dpkg-divert --local --rename --add /sbin/initctl \
	&& cp -a /usr/sbin/policy-rc.d /sbin/initctl \
	&& sed -i 's/^exit.*/exit 0/' /sbin/initctl \
	\
# https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap#L71-L78
	&& echo 'force-unsafe-io' > /etc/dpkg/dpkg.cfg.d/docker-apt-speedup \
	\
# https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap#L85-L105
	&& echo 'DPkg::Post-Invoke { "rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true"; };' > /etc/apt/apt.conf.d/docker-clean \
	&& echo 'APT::Update::Post-Invoke { "rm -f /var/cache/apt/archives/*.deb /var/cache/apt/archives/partial/*.deb /var/cache/apt/*.bin || true"; };' >> /etc/apt/apt.conf.d/docker-clean \
	&& echo 'Dir::Cache::pkgcache ""; Dir::Cache::srcpkgcache "";' >> /etc/apt/apt.conf.d/docker-clean \
	\
# https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap#L109-L115
	&& echo 'Acquire::Languages "none";' > /etc/apt/apt.conf.d/docker-no-languages \
	\
# https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap#L118-L130
	&& echo 'Acquire::GzipIndexes "true"; Acquire::CompressionTypes::Order:: "gz";' > /etc/apt/apt.conf.d/docker-gzip-indexes \
	\
# https://github.com/docker/docker/blob/9a9fc01af8fb5d98b8eec0740716226fadb3735c/contrib/mkimage/debootstrap#L134-L151
	&& echo 'Apt::AutoRemove::SuggestsImportant "false";' > /etc/apt/apt.conf.d/docker-autoremove-suggests

# verify that the APT lists files do not exist
RUN [ -z "$(apt-get indextargets)" ]
# (see https://bugs.launchpad.net/cloud-images/+bug/1699913)

# make systemd-detect-virt return "docker"
# See: https://github.com/systemd/systemd/blob/aa0c34279ee40bce2f9681b496922dedbadfca19/src/basic/virt.c#L434
RUN mkdir -p /run/systemd && echo 'docker' > /run/systemd/container


# apt-get installs
ARG DEBIAN_FRONTEND=noninteractive
RUN apt-get -qq update && \
    apt-get -qq install -y --no-install-recommends autoconf wget git curl build-essential libbz2-dev zlib1g-dev liblzma-dev libeigen3-dev libreadline-gplv2-dev libncursesw5-dev libssl-dev libsqlite3-dev tk-dev libgdbm-dev libc6-dev libcurl4-openssl-dev ca-certificates python3.7-dev python3-pip python3.7-venv && \
    apt-get -qq clean && \
    apt-get -qq purge

# install cmake
WORKDIR /root
RUN mkdir /root/cmake && \
    wget -q https://cmake.org/files/v3.17/cmake-3.17.0-Linux-x86_64.sh --no-check-certificate && \
    sh /root/cmake-3.17.0-Linux-x86_64.sh --prefix=/root/cmake --skip-license && \
    ln -s /root/cmake/bin/cmake /usr/local/bin/cmake

# htslib
WORKDIR /root/
RUN wget -q https://github.com/samtools/htslib/releases/download/1.9/htslib-1.9.tar.bz2 --no-check-certificate && \
    tar -vxjf htslib-1.9.tar.bz2 && \
    rm htslib-1.9.tar.bz2 && \
    cd /root/htslib-1.9 && \
    ./configure --prefix /usr/local --enable-plugins CPPFLAGS="-fPIC" CFLAGS="-fPIC" && \
    make && \
    make install

# boost install
WORKDIR /root/
RUN wget -q -O boost_1_69_0.tar.gz https://sourceforge.net/projects/boost/files/boost/1.69.0/boost_1_69_0.tar.gz/download --no-check-certificate && \
    tar -xzf boost_1_69_0.tar.gz >/dev/null && \
    rm boost_1_69_0.tar.gz && \
    cd /root/boost_1_69_0/ && \
    ./bootstrap.sh --with-libraries=system,date_time,filesystem,iostreams,coroutine,context,regex,thread,atomic >/dev/null && \
    ./b2 -d0 cxxflags="-fPIC" cflags="-fPIC" link=static -a && \
    ./b2 -d0 install

# hdf5 install
WORKDIR /root/
RUN wget -q https://support.hdfgroup.org/ftp/HDF5/releases/hdf5-1.10/hdf5-1.10.4/src/hdf5-1.10.4.tar.gz && \
    tar -xzf hdf5-1.10.4.tar.gz && \
    cd hdf5-1.10.4 && \
    ./configure --enable-threadsafe --disable-hl --prefix=/usr/local/ && \
    make && \
    make install

## setup python
WORKDIR /root/
RUN curl https://bootstrap.pypa.io/get-pip.py -o get-pip.py && \
    python3.7 get-pip.py && \
    python3.7 -m pip -q install setuptools cython virtualenv


CMD ["/bin/bash"]
