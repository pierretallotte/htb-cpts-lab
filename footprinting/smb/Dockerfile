FROM debian

RUN apt-get update && apt-get upgrade -y
RUN apt-get install -y samba

RUN useradd -m john
RUN useradd -m mary

RUN (echo "smbpass" ; echo "smbpass") | smbpasswd -a -s mary

# This folder will be used in the modified configuration of smb
RUN mkdir /smbshare

RUN touch /var/log/samba/log.txt
CMD /usr/sbin/smbd && tail -f /var/log/samba/log.txt