FROM debian

RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y openssh-server

# "john" user has the password "john"
RUN useradd -m john
RUN echo "john:pass" | chpasswd

# "mary" user has no password
RUN useradd -m mary
RUN passwd -d mary

# The password of the root user is "root"
RUN echo "root:root" | chpasswd

EXPOSE 22

CMD ["service", "ssh", "start", "-D"]