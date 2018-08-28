# base image
FROM frolvlad/alpine-oraclejdk8:slim

# MAINTAINER
MAINTAINER wanghy@wfy.com.cn

#RUN yum -y update

# elastic-job-lite-console-2.1.5.tar.gz
ADD elastic-job-lite-console-2.1.5.tar.gz /usr/local

# work dir
WORKDIR /usr/local/elastic-job-lite-console-2.1.5/bin

ENV PATH $PATH:/usr/local/elastic-job-lite-console-2.1.5/bin

RUN sh -c 'touch ./start.sh'
RUN sh -c 'chmod +x ./start.sh'
#RUN sh -c './start.sh'

EXPOSE 8899

ENTRYPOINT ["/bin/sh","start.sh"]
