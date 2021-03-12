
ARG UBUNTU_VERSION=18.04
FROM ubuntu:${UBUNTU_VERSION}

RUN apt-get update
RUN apt-get install -y git  && \
    apt-get install -y wget && \
    apt-get install -y rsync && \
    apt-get install -y curl

RUN wget https://golang.org/dl/go1.15.9.linux-amd64.tar.gz && \
    rm -rf /usr/local/go && tar -C /usr/local -xvf go1.15.9.linux-amd64.tar.gz && \
    echo "export PATH=$PATH:/usr/local/go/bin" >> /root/.bashrc && \
    /bin/bash -c "source /root/.bashrc"

RUN wget https://github.com/gohugoio/hugo/releases/download/v0.81.0/hugo_0.81.0_Linux-64bit.deb && \
    dpkg -i hugo_0.81.0_Linux-64bit.deb

RUN mkdir /vitah
WORKDIR /vitah/blog

EXPOSE 1313

CMD ["/bin/bash"]