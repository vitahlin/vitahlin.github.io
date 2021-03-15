ARG UBUNTU_VERSION=18.04
FROM ubuntu:${UBUNTU_VERSION}

RUN apt-get update
RUN apt-get install -y git wget rsync

ARG GO_VERSION=1.16.2
ARG HUGO_VERSION=0.81.0

RUN wget https://golang.org/dl/go${GO_VERSION}.linux-amd64.tar.gz && \
    rm -rf /usr/local/go && tar -C /usr/local -xvf go${GO_VERSION}.linux-amd64.tar.gz && \
    echo "export PATH=$PATH:/usr/local/go/bin" >> /root/.bashrc && \
    /bin/bash -c "source /root/.bashrc"

RUN wget https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_${HUGO_VERSION}_Linux-64bit.deb && \
    dpkg -i hugo_${HUGO_VERSION}_Linux-64bit.deb

RUN wget https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_Linux-64bit.deb && \
    dpkg -i hugo_extended_${HUGO_VERSION}_Linux-64bit.deb

RUN mkdir /vitah
WORKDIR /vitah

COPY init.sh /vitah/

EXPOSE 1313

CMD ["/bin/bash"]