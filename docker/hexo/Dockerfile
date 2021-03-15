
ARG NODE_VERSION=11.1.0
FROM node:${NODE_VERSION}-alpine

ARG user_name=vitahlin
ARG user_mail=linw1225@gmail.com

RUN apk add --no-cache bash git openssh && \
    git --version && \
    git config  --global user.name ${user_name} && \
    git config  --global user.email ${user_mail}

RUN npm install -g hexo-cli

RUN mkdir /vitah
WORKDIR /vitah/hexo

CMD ["/bin/sh"]