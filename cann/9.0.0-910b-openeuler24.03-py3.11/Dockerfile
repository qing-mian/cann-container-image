# Stage 1: Install Python
FROM docker.1ms.run/openeuler/openeuler:24.03 AS python-installer

# Python Environment variables
ENV PATH=/usr/local/python3.11.14/bin:${PATH}

RUN yum update -y && \
    yum install -y \
        gcc \
        gcc-c++ \
        make \
        cmake \
        curl \
        zlib-devel \
        bzip2-devel \
        openssl-devel \
        ncurses-devel \
        sqlite-devel \
        readline-devel \
        tk-devel \
        gdbm-devel \
        libpcap-devel \
        xz-devel \
        libev-devel \
        expat-devel \
        libffi-devel \
        systemtap-sdt-devel \
    && yum clean all \
    && rm -rf /var/cache/yum \
    && rm -rf /tmp/*

# Install Python
RUN curl -fsSL https://repo.huaweicloud.com/python/3.11.14/Python-3.11.14.tgz -o /tmp/Python-3.11.14.tgz && \
    tar -xf /tmp/Python-3.11.14.tgz -C /tmp && \
    cd /tmp/Python-3.11.14 && \
    mkdir -p /usr/local/python3.11.14/lib && \
    ./configure --enable-shared --enable-shared LDFLAGS="-Wl,-rpath /usr/local/python3.11.14/lib" --prefix=/usr/local/python3.11.14 && \
    make -j $(nproc) && \
    make altinstall && \
    ln -sf /usr/local/python3.11.14/bin/python3.11 /usr/local/python3.11.14/bin/python3 && \
    ln -sf /usr/local/python3.11.14/bin/pip3.11 /usr/local/python3.11.14/bin/pip3 && \
    ln -sf /usr/local/python3.11.14/bin/python3 /usr/local/python3.11.14/bin/python && \
    ln -sf /usr/local/python3.11.14/bin/pip3 /usr/local/python3.11.14/bin/pip && \
    rm -rf /tmp/*

# Stage 2: Install CANN
FROM python-installer AS cann-installer

ARG TARGETPLATFORM

RUN yum update -y && \
    yum install -y \
        unzip \
        pciutils \
        net-tools \
        lapack-devel \
        gcc-gfortran \
        util-linux \
        findutils \
        wget \
    && yum clean all \
    && rm -rf /var/cache/yum

# Note: Install CANN runtime dependencies
RUN pip install --no-cache-dir --upgrade pip && \
    pip install torch_npu==2.6.0 pyyaml numpy decorator sympy scipy attrs protobuf psutil

# Note: Get the download link according to ARCH and download the installation package
RUN ARCH=$(case "${TARGETPLATFORM}" in \
        "linux/amd64") echo "x86_64" ;; \
        "linux/arm64") echo "aarch64" ;; \
        *) echo "Unsupported TARGETPLATFORM: ${TARGETPLATFORM}" && exit 1 ;; \
    esac) && \
    CANN_TOOLKIT_URL=https://ascend.devcloud.huaweicloud.com/artifactory/cann-run-release/software/master/20260213000325157/aarch64/Ascend-cann-910b-ops_9.0.0_linux-${ARCH}.run && \
    CANN_OPS_URL=https://ascend.devcloud.huaweicloud.com/artifactory/cann-run-release/software/master/20260213000325157/aarch64/Ascend-cann-910b-ops_9.0.0_linux-${ARCH}.run && \
    wget --quiet --header="Referer: https://www.hiascend.com/" ${CANN_TOOLKIT_URL} -O ~/Ascend-cann-toolkit.run && \
    wget --quiet --header="Referer: https://www.hiascend.com/" ${CANN_OPS_URL} -O ~/Ascend-cann-ops.run

# Note: Install CANN Toolkit Development Kit Package
RUN chmod +x ~/Ascend-cann-toolkit.run && \
    ~/Ascend-cann-toolkit.run --quiet --install --install-for-all && \
    rm -f ~/Ascend-cann-toolkit.run

# Note: Install CANN Ops Operator Package
RUN chmod +x ~/Ascend-cann-ops.run && \
    ~/Ascend-cann-ops.run --quiet --install --install-for-all && \
    rm -f ~/Ascend-cann-ops.run
    
# Stage 3: Copy results from previous stages
FROM docker.1ms.run/openeuler/openeuler:24.03 AS official-openeuler

ARG TARGETPLATFORM

# Python Environment variables
ENV PATH=/usr/local/python3.11.14/bin:${PATH}

SHELL [ "/bin/bash", "-c" ]

RUN yum update -y && \
    yum install -y \
        ca-certificates \
        bash \
        glibc \
        sqlite-devel \
        gcc \
        gcc-c++ \
        make \
        cmake \
        git \
        vim \
        wget \
        jq \
        curl \
        numactl-devel \
    && yum clean all \
    && rm -rf /var/cache/yum \
    && rm -rf /tmp/*

COPY --from=cann-installer /usr/local/python3.11.14 /usr/local/python3.11.14
COPY --from=cann-installer /usr/local/Ascend /usr/local/Ascend
COPY --from=cann-installer /etc/Ascend /etc/Ascend

# Note: Set environment variables
RUN \
    ARCH=$(case "${TARGETPLATFORM}" in \
        "linux/amd64") echo "x86_64" ;; \
        "linux/arm64") echo "aarch64" ;; \
        *) echo "Unsupported TARGETPLATFORM: ${TARGETPLATFORM}" && exit 1 ;; \
    esac) && \
    ln -sf /usr/include/c++/12/${ARCH}-openEuler-linux /usr/include/c++/12/${ARCH}-linux-gnu && \
    CANN_TOOLKIT_ENV_FILE="/usr/local/Ascend/cann/set_env.sh" && \
    CANN_BISHENG_ENV_FILE="/usr/local/Ascend/cann/share/info/ascendnpu-ir/bin/set_env.sh" && \
    echo "source ${CANN_TOOLKIT_ENV_FILE}" >> /etc/profile && \
    echo "source ${CANN_TOOLKIT_ENV_FILE}" >> ~/.bashrc && \
    echo "source ${CANN_BISHENG_ENV_FILE}" >> /etc/profile && \
    echo "source ${CANN_BISHENG_ENV_FILE}" >> ~/.bashrc


RUN \
    git clone -b ci-test https://gitcode.com/chenkunjie0506/Mooncake.git && \
    cd Mooncake && \
    bash scripts/ascend/dependencies_ascend_installation.sh

ENTRYPOINT ["/bin/bash", "-c", "\
    source /usr/local/Ascend/cann/set_env.sh && \
    source /usr/local/Ascend/cann/share/info/ascendnpu-ir/bin/set_env.sh && \
    exec \"$@\"", "--"]


