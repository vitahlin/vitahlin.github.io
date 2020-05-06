
ARG UBUNTU_VERSION=18.04
FROM ubuntu:${UBUNTU_VERSION}

RUN apt-get update && \
    apt-get install hugo && \
    hugo version

RUN mkdir /vitah
WORKDIR /vitah/hexo

CMD ["/bin/bash"]