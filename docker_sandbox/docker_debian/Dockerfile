#FROM debian:10
FROM debian:11-slim

LABEL   maintainer="dispelk9@gmail.com"
ENV     ANSIBLE_VERSION 2.9.17

RUN apt update

RUN apt install sudo locales --yes 

RUN apt install -y python3
RUN apt install -y python3-pip
    
RUN pip3 install --upgrade pip;                 \
    pip3 install "ansible==${ANSIBLE_VERSION}"; \
    pip3 install ansible

RUN apt install -y git

COPY text /tmp/test