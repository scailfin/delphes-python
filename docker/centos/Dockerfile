ARG BUILDER_IMAGE=neubauergroup/centos-build-base:3.8.11
FROM neubauergroup/centos-root-base:6.24.00 as root_cern_image
FROM ${BUILDER_IMAGE} as builder

USER root
WORKDIR /

SHELL ["/bin/bash", "-o", "pipefail", "-c"]

COPY --from=root_cern_image /usr/local/root-cern /usr/local/root-cern
COPY --from=root_cern_image /tmp/packages.txt /tmp/packages.txt
COPY --from=root_cern_image /tmp/requirements.txt /tmp/requirements.txt
ENV PYTHONPATH=/usr/local/root-cern/lib:$PYTHONPATH
ENV LD_LIBRARY_PATH=/usr/local/root-cern/lib:$LD_LIBRARY_PATH
ENV ROOTSYS=/usr/local/root-cern
ENV PATH="${PATH}:${ROOTSYS}/bin"

# hadolint ignore=SC2046
RUN yum update -y && \
    yum install -y epel-release && \
    yum install -y $(cat /tmp/packages.txt) && \
    yum install -y \
      gcc \
      gcc-c++ \
      gcc-gfortran \
      make \
      vim \
      zlib \
      zlib-devel \
      bzip2 \
      bzip2-devel \
      rsync \
      bash-completion \
      bash-completion-extras \
      wget \
      git && \
    yum clean all && \
    yum autoremove -y && \
    python -m pip --no-cache-dir install --upgrade pip setuptools wheel && \
    python -m pip --no-cache-dir install --requirement /tmp/requirements.txt && \
    python -m pip list

# Install HepMC
ARG HEPMC_VERSION=2.06.11
RUN mkdir /code && \
    cd /code && \
    wget http://hepmc.web.cern.ch/hepmc/releases/hepmc${HEPMC_VERSION}.tgz && \
    tar xvfz hepmc${HEPMC_VERSION}.tgz && \
    mv HepMC-${HEPMC_VERSION} src && \
    source scl_source enable devtoolset-8 && \
    cmake \
      -DCMAKE_CXX_COMPILER=$(command -v g++) \
      -DCMAKE_BUILD_TYPE=Release \
      -Dbuild_docs:BOOL=OFF \
      -Dmomentum:STRING=MEV \
      -Dlength:STRING=MM \
      -DCMAKE_INSTALL_PREFIX=/usr/local/venv \
      -S src \
      -B build && \
    cmake build -L && \
    cmake --build build -- -j$(($(nproc) - 1)) && \
    cmake --build build --target install && \
    rm -rf /code

# Install FastJet
ARG FASTJET_VERSION=3.3.4
RUN mkdir /code && \
    cd /code && \
    wget http://fastjet.fr/repo/fastjet-${FASTJET_VERSION}.tar.gz && \
    tar xvfz fastjet-${FASTJET_VERSION}.tar.gz && \
    cd fastjet-${FASTJET_VERSION} && \
    source scl_source enable devtoolset-8 && \
    ./configure --help && \
    export CXX=$(command -v g++) && \
    export PYTHON=$(command -v python3) && \
    export PYTHON_CONFIG=$(find /usr/local/ -iname "python-config.py") && \
    ./configure \
      --prefix=/usr/local/venv \
      --enable-pyext=yes && \
    make -j$(($(nproc) - 1)) && \
    make check && \
    make install && \
    rm -rf /code

# Install LHAPDF
ARG LHAPDF_VERSION=6.3.0
RUN mkdir /code && \
    cd /code && \
    wget https://lhapdf.hepforge.org/downloads/?f=LHAPDF-${LHAPDF_VERSION}.tar.gz -O LHAPDF-${LHAPDF_VERSION}.tar.gz && \
    tar xvfz LHAPDF-${LHAPDF_VERSION}.tar.gz && \
    cd LHAPDF-${LHAPDF_VERSION} && \
    source scl_source enable devtoolset-8 && \
    ./configure --help && \
    export CXX=$(command -v g++) && \
    export PYTHON=$(command -v python3) && \
    ./configure \
      --prefix=/usr/local/venv && \
    make -j$(($(nproc) - 1)) && \
    make install && \
    rm -rf /code

# Install PYTHIA
ARG PYTHIA_VERSION=8243
# PYTHON_VERSION already exists in the base image
# CentOS 7 gcc v4.8.5 is old enough need to specify -std=c++11
RUN mkdir /code && \
    cd /code && \
    wget "https://pythia.org/download/pythia${PYTHIA_VERSION:0:2}/pythia${PYTHIA_VERSION}.tgz" && \
    tar xvfz pythia${PYTHIA_VERSION}.tgz && \
    cd pythia${PYTHIA_VERSION} && \
    source scl_source enable devtoolset-8 && \
    ./configure --help && \
    export PYTHON_MINOR_VERSION=${PYTHON_VERSION::3} && \
    ./configure \
      --prefix=/usr/local/venv \
      --arch=Linux \
      --cxx=g++ \
      --enable-64bit \
      --with-gzip \
      --with-hepmc2=/usr/local/venv \
      --with-lhapdf6=/usr/local/venv \
      --with-fastjet3=/usr/local/venv \
      --with-python-bin=/usr/local/venv/bin/ \
      --with-python-lib=/usr/local/venv/lib/python${PYTHON_MINOR_VERSION} \
      --with-python-include=/usr/local/include/python${PYTHON_MINOR_VERSION} \
      --cxx-common="-O2 -m64 -pedantic -W -Wall -Wshadow -fPIC -std=c++11" \
      --cxx-shared="-shared -std=c++11" && \
    make -j$(($(nproc) - 1)) && \
    make install && \
    rm -rf /code

ARG DELPHES_VERSION=3.5.0
# hadolint ignore=SC1091
# hadolint ignore=SC2046
RUN mkdir -p /code && \
    cd /code && \
    git clone --depth 1 https://github.com/delphes/delphes.git \
      --branch "${DELPHES_VERSION}" \
      --single-branch && \
    source scl_source enable devtoolset-8 && \
    cmake \
      -DCMAKE_INSTALL_PREFIX=/usr/local/venv \
      -DPYTHIA8_DATA=/usr/local/venv/share/Pythia8 \
      -S delphes \
      -B build && \
    cmake build -L && \
    cmake --build build -- -j$(($(nproc) - 1)) && \
    cmake --build build --target install && \
    rm -rf /code

# Create non-root user "docker"
RUN useradd --shell /bin/bash -m docker && \
   cp /root/.bashrc /home/docker/ && \
   cp /root/.bash_profile /home/docker/ && \
   mkdir /home/docker/data && \
   chown -R --from=root docker /home/docker && \
   chown -R --from=root docker /usr/local/venv && \
   chown -R --from=root docker /usr/local/root-cern

# Use en_US.utf8 locale to avoid issues with ASCII encoding
# as C.UTF-8 not available on CentOS 7
ENV LC_ALL=en_US.utf8
ENV LANG=en_US.utf8

ENV PYTHONPATH=/usr/local/venv/lib:${PYTHONPATH}
ENV LD_LIBRARY_PATH=/usr/local/venv/lib:$LD_LIBRARY_PATH

# Default user is root to avoid uid write permission problems with volumes
ENV HOME /root
WORKDIR ${HOME}/data

ENTRYPOINT ["/bin/bash", "-l", "-c"]
CMD ["/bin/bash"]
