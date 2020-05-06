
ARG UBUNTU_VERSION=18.04
FROM ubuntu:${UBUNTU_VERSION}

RUN apt-get update && \
    apt-get install -y git && \
    apt-get install -y wget

RUN wget https://github.com/gohugoio/hugo/releases/download/v0.69.2/hugo_0.69.2_Linux-64bit.deb && \
    dpkg -i hugo_0.69.2_Linux-64bit.deb && \
    hugo version

RUN mkdir /vitah
WORKDIR /vitah/blog

EXPOSE 1313

CMD ["/bin/bash"]