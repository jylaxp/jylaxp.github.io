---
layout: post
title: Dockerfile示例
---

# sshd
```sshd
FROM ubuntu:latest
MAINTAINER from jyl

RUN echo "deb http://mirrors.aliyun.com/ubuntu/ vivid main restricted universe multiverse" > /etc/apt/sources.list
RUN echo "deb http://mirrors.aliyun.com/ubuntu/ vivid-security main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://mirrors.aliyun.com/ubuntu/ vivid-updates main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://mirrors.aliyun.com/ubuntu/ vivid-proposed main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb http://mirrors.aliyun.com/ubuntu/ vivid-backports main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb-src http://mirrors.aliyun.com/ubuntu/ vivid main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb-src http://mirrors.aliyun.com/ubuntu/ vivid-security main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb-src http://mirrors.aliyun.com/ubuntu/ vivid-updates main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb-src http://mirrors.aliyun.com/ubuntu/ vivid-proposed main restricted universe multiverse" >> /etc/apt/sources.list
RUN echo "deb-src http://mirrors.aliyun.com/ubuntu/ vivid-backports main restricted universe multiverse" >> /etc/apt/sources.list

RUN apt-get update
RUN apt-get install -y openssh-server
RUN mkdir -p /var/run/sshd
RUN mkdir -p /root/.ssh
RUN sed -ri 's/(session[ \t]*required[ \t]*pam_loginuid.so)/#\1/g' /etc/pam.d/sshd

ADD authorized_keys /root/.ssh/authorized_keys
ADD run.sh /run.sh
RUN chmod 755 /run.sh

EXPOSE 22

CMD ["/run.sh"]
```
# jdk
```jdk
FROM ubuntu:latest
MAINTAINER  jiang

ADD jdk-8u144-linux-x64.tar.gz /opt/

ENV JAVA_HOME=/opt/jdk1.8.0_144
ENV JRE_HOME=${JAVA_HOME}/jre
ENV CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
ENV PATH=${JAVA_HOME}/bin:$PATH
```