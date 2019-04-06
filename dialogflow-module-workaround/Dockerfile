FROM amazonlinux:latest
MAINTAINER nakamura <xxxxxxx@classmethod.jp>
RUN yum -y update
# Node.js
RUN yum -y install gcc-c++ make wget
RUN curl --silent --location https://rpm.nodesource.com/setup_8.x | bash -
RUN yum -y install nodejs
# typescript
RUN npm -g install typescript
# tar gzip
RUN yum install -y tar.x86_64
RUN yum install -y gzip
# Apex
RUN curl https://raw.githubusercontent.com/apex/apex/master/install.sh | sh
