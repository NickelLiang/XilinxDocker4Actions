ARG VERSION_NUM=2022.1
ARG WEB_INSTALLER=Xilinx_Unified_2022.1_0420_0327_Lin64.bin
ARG INSTALL_CONFIG=install_config.txt
ARG AUTH_KEY=wi_authentication_key

## Run installer in separate image to minimize final image size
FROM ubuntu:22.04 AS installation
## Refresh arguments
ARG WEB_INSTALLER
ARG INSTALL_CONFIG
ARG AUTH_KEY
ARG VERSION_NUM

COPY ${WEB_INSTALLER} /${WEB_INSTALLER}
COPY ${INSTALL_CONFIG} /${INSTALL_CONFIG}
COPY ${AUTH_KEY} /root/.Xilinx/wi_authentication_key

# Required packages for xilinx tools
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git \
                       device-tree-compiler \
                       default-jre \
                       openjdk-11-jdk && \
    apt-get autoclean && apt-get autoremove && \
    rm -rf /var/lib/apt/lists/*

## Xilinx 2022.1 will include versal support in the following folder regardless of your install config
## Remove this can save up to ~50G of disk space
## https://support.xilinx.com/s/question/0D52E00006ln0TnSAI/question-regarding-vivado-20211-installation-size-and-device-support?language=en_US
RUN /${WEB_INSTALLER} --noexec --target ./installer && \
    cd installer && \
	./xsetup \
  		--agree XilinxEULA,3rdPartyEULA \
  		--config /${INSTALL_CONFIG} \
  		--batch INSTALL && \
    cd .. && \
    rm -rf /tools/Xilinx/.xinstall && \
    rm /${WEB_INSTALLER} && \
    rm /${INSTALL_CONFIG} && \
    rm /root/.Xilinx/wi_authentication_key && \
    rm -rf /tools/Xilinx/Vivado/${VERSION_NUM}/data/parts/xilinx/devint/vault/versal

## Create Vivado Image
FROM ubuntu:22.04 AS vivado
## Refresh arguments
ARG VERSION_NUM

## cs_server needs the UTF-8 env vars
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
COPY --from=installation /tools/Xilinx/Vivado /tools/Xilinx/Vivado

## support packages for xilinx tools
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git \
                       locales-all \
                       device-tree-compiler \
                       default-jre \
                       openjdk-11-jdk && \
    apt-get autoclean && apt-get autoremove && \
    rm -rf /var/lib/apt/lists/* && \
    ## workaround for libtinfo.so.5 dependency of vivado
    cd /lib/x86_64-linux-gnu && ln -s libtinfo.so.6 libtinfo.so.5 && \
    echo "source /tools/Xilinx/Vivado/${VERSION_NUM}/settings64.sh" >> /etc/bash.bashrc

## Create Vitis Image
FROM ubuntu:22.04 AS vitis
## Refresh arguments
ARG VERSION_NUM

# cs_server needs the UTF-8 env vars
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
COPY --from=installation /tools/Xilinx/Vitis /tools/Xilinx/Vitis

# support packages for xilinx tools
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y git \
                       locales-all \
                       device-tree-compiler \
                       default-jre \
                       openjdk-11-jdk && \
    apt-get autoclean && apt-get autoremove && \
    rm -rf /var/lib/apt/lists/* && \
    # workaround for libtinfo.so.5 dependency of vitis
    cd /lib/x86_64-linux-gnu && ln -s libtinfo.so.6 libtinfo.so.5 && \
    echo "source /tools/Xilinx/Vitis/${VERSION_NUM}/settings64.sh" >> /etc/bash.bashrc