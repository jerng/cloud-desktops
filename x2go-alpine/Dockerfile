FROM alpine:3.19
RUN apk add --no-cache \
    openssh openssh-server openssh-keygen \
    x2goserver coreutils mesa-dri-gallium \
    lxqt-desktop lximage-qt obconf-qt pavucontrol-qt screengrab arandr sddm
    # (coreutils) is needed for (readlink), used by (x2gostartagent) as the
    # (busybox) version has incompatible flags
RUN sed -ri 's/X11Forwarding no/X11Forwarding yes/g' /etc/ssh/sshd_config \
    && sed -ri 's/#X11DisplayOffset 10/X11DisplayOffset 10/g' /etc/ssh/sshd_config \
    && sed -ri 's/#X11UseLocalhost yes/X11Uselocalhost yes/g' /etc/ssh/sshd_config
COPY id_rsa.pub /root/.ssh/authorized_keys
EXPOSE 22
ENTRYPOINT ["sh", "-xc", "ssh-keygen -A && /usr/sbin/sshd -D"]

# from :
# https://stackoverflow.com/questions/77459380/i-cant-connect-to-my-alpine-docker-image-running-an-ssh-server-with-rsa-keys
#
# docker build -t test .
# docker run -d --rm -p 127.0.0.1:10022:22 --name test test
# ssh -i key -p 10022 -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -o batchmode=yes root@127.0.0.1 echo hello world
