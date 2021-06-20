ARG BUILDER_IMAGE=neubauergroup/centos-build-base:latest
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
      git && \
    yum clean all && \
    yum autoremove -y && \
    python -m pip --no-cache-dir install --upgrade pip setuptools wheel && \
    python -m pip --no-cache-dir install --requirement /tmp/requirements.txt && \
    python -m pip list

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
      -S delphes \
      -B build && \
    cmake build -L && \
    cmake --build build -- -j$(($(nproc) - 1)) && \
    cmake --build build --target install && \
    rm -rf /code

RUN chown -R --from=root docker /usr/local/root-cern && \
    chown -R --from=root docker /home/docker && \
    chown -R --from=root docker /home/docker/.mg5

# Default user is root to avoid uid write permission problems with volumes
ENV HOME /root
WORKDIR ${HOME}/data

ENTRYPOINT ["/bin/bash", "-l", "-c"]
CMD ["/bin/bash"]
