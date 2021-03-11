
ARG UBUNTU_VERSION=18.04
FROM ubuntu:${UBUNTU_VERSION}

RUN apt-get update && \
    apt-get install -y git && \
    apt-get install -y wget

RUN wget https://github.com/gohugoio/hugo/releases/download/v0.81.0/hugo_0.81.0_Linux-64bit.deb && \
    dpkg -i hugo_0.81.0_Linux-64bit.deb

RUN mkdir /vitah
WORKDIR /vitah/blog

EXPOSE 1313

CMD ["/bin/bash"]