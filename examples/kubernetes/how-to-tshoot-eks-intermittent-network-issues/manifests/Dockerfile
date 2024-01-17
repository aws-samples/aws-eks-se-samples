FROM amazon/aws-cli:latest

RUN yum install tcpdump wget -y
RUN groupadd --gid 1000 nonroot 
RUN useradd --uid 1000 --gid 1000 nonroot
RUN echo 'nonroot ALL=(ALL) NOPASSWD: ALL' >> /etc/sudoers 
RUN chgrp 1000 /usr/sbin/tcpdump 
RUN chmod 750 /usr/sbin/tcpdump 
RUN setcap cap_net_admin,cap_net_raw+eip /usr/sbin/tcpdump
