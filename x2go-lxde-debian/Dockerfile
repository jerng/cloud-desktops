FROM debian:10.10 

LABEL "author"="jerng" \
      "architecture"="amd64" \
      "approximate_date_modified"="2021-06" \
      "purpose_of_this_image"="Runs a Linux Desktop Server which you can \
connect to remotely, via a network." \
      "based_on"="https://github.com/tatsuya6502/docker-x2go" \
      "core_technologies_used"="linux (debian), lxde, x2go" \
      "tested_on_infrastructures"="localhost; \
AWS S3 > CodeBuild > Elastic Container Repository > Fargate" \
      "security_advice"="This build is not secure; use it at your own \
risk; you may begin by taking further advice from AWS ECR's free scanning \
service." \
      "comment"="X2Go uses the NX3 protocol which seems to deliver choppy \
audio via PulseAudio when the server is hosted on a public cloud, and \
the client is far away, whereas it is open-source; NoMachine uses the \
NX4 protocol which seems to deliver smooth audio under similar \
infrastructure, whereas it is freemium" \
      "public_repository"="https://hub.docker.com/r/jerng/x2go-lxde-debian"

ENV DEBIAN_FRONTEND noninteractive

# Update repos
RUN apt update -y

# Install requirements
RUN apt install -y \
            #
        software-properties-common apt-utils \
            # for apt
            #
        htop nload sudo vim screen \
            # for admin
            #
        openssh-server \
            # for ssh
            #
        lxde lxde-common \
            # for window manager
            #
        dirmngr gnupg \
            # for apt-key for (x2go) 
            #
        wget \
            # for (google-chrome)
            #
    && apt remove -y firefox-esr
        # redundant

# Decrunking (crunk : you get an "Error: No session for pid NNN" on LXDE startup)
RUN mv /usr/bin/lxpolkit /usr/bin/lxpolkit.ORIG

# Upgrade packages
RUN apt upgrade -y

# Install X2Go server components
RUN apt-key adv --recv-keys --keyserver keys.gnupg.net E1F958385BFE2B6E \
    && sed -i '1s/^/deb http:\/\/packages.x2go.org\/debian stretch extras main\n/' /etc/apt/sources.list \
    && sed -i '1s/^/deb-src http:\/\/packages.x2go.org\/debian stretch extras main\n/' /etc/apt/sources.list \
    && apt update -y \
    && apt install -y x2goserver x2goserver-xsession --no-install-recommends

# If you are on arm64, use (chromium) instead of (google-chrome)
RUN wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb \
    && apt install -y ./google-chrome-stable_current_amd64.deb \
    && rm ./google-chrome-stable_current_amd64.deb \
        #
    && sed -i 's/exec -a "$0" "$HERE\/chrome" "$@"/exec -a "$0" "$HERE\/google-chrome-stable" "$@" --no-sandbox/' /usr/bin/google-chrome \
    && sed -i 's/google-chrome-stable %U/google-chrome-stable %U --no-sandbox/' /usr/share/applications/google-chrome.desktop 
        # Decrunking (crunk: chromium, or google-chrome, won't start)

# Configure SSH
RUN mkdir -p /var/run/sshd \
    && sed -ri 's/UseDNS yes/#UseDNS yes/g' /etc/ssh/sshd_config\
    && echo "UseDNS no" >> /etc/ssh/sshd_config \
    #     && sed -ri 's/UsePAM yes/#UsePAM yes/g' /etc/ssh/sshd_config \
    #     && sed -ri 's/#UsePAM no/UsePAM no/g' /etc/ssh/sshd_config \
    && sed -i "s/.*RSAAuthentication.*/RSAAuthentication yes/g" /etc/ssh/sshd_config \
    && sed -i "s/.*PubkeyAuthentication.*/PubkeyAuthentication yes/g" /etc/ssh/sshd_config \
    && sed -i "s/.*PasswordAuthentication.*/PasswordAuthentication no/g" /etc/ssh/sshd_config \
    && sed -i "s/.*AuthorizedKeysFile.*/AuthorizedKeysFile\t\.ssh\/authorized_keys/g" /etc/ssh/sshd_config \
    && sed -i "s/.*PermitRootLogin.*/PermitRootLogin no/g" /etc/ssh/sshd_config

# Create a user
RUN useradd -m docker \
    && mkdir -p /home/docker/.ssh \
    && chmod 700 /home/docker/.ssh \
    && chown docker:docker /home/docker/.ssh \
    && mkdir -p /etc/sudoers.d \
    && echo "docker ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

# Allow "ssh hostname sudo <cmd>" without -t (TTY) option,
# because some remote management tools such as Capistrano
# and Vagrant require it.
    #
    # IMPORTANT: The user must have "NOPASSWORD: ALL" because
    # "ssh hostname sudo <cmd>" will show the password in clear.
    # If not, you should run "ssh -t hostname sudo <cmd>" instead.
    #
# RUN echo 'docker        ALL=(ALL)       NOPASSWD: ALL\nDefaults:worker !requiretty' > /etc/sudoers.d/999-sudoers-docker \
#    && chmod 440 /etc/sudoers.d/999-sudoers-docker

# Startup script - A - (you may be operating docker manually) - generates a key on "docker run" which can be printed with "docker logs CONTAINER_NAME"
    #
    # Create ssh "client" key\n\
    # http://stackoverflow.com/a/20977657\n\
# RUN echo '#!/bin/sh\n\
#     \n\
#     \n\
#     USER_HOME=/home/docker\n\
#     \n\
#     KEYGEN=/usr/bin/ssh-keygen\n\
#     KEYFILE=${USER_HOME}/.ssh/id_rsa\n\
#     \n\
#     if [ ! -f $KEYFILE ]; then\n\
#         $KEYGEN -q -t rsa -N "" -f $KEYFILE\n\
#         cat $KEYFILE.pub >> ${USER_HOME}/.ssh/authorized_keys\n\
#     fi\n\
#     \n\
#     echo "== Use this private key to log in =="\n\
#     cat $KEYFILE\n\
#     \n\
#     /usr/sbin/sshd -D' > /root/start-sshd.sh

# Startup script - B - (you may be automating docker) - imports an SSH key from an environmental variable
RUN echo '#!/bin/sh \n\
    echo $SSH_PUBLIC_KEY > /home/docker/.ssh/authorized_keys \n\
    /usr/sbin/sshd -D' > /root/start-sshd.sh

# Mode the startup script
RUN chmod 744 /root/start-sshd.sh

# Set time
RUN ln -sf /usr/share/zoneinfo/Asia/Kuala_Lumpur /etc/localtime

# Set .bashrc for stylistic preferences
RUN usermod --shell /bin/bash docker \
    && echo "export PATH=\"\$HOME:\$PATH\"\n\
\n\
\n\
    case \$- in\n\
        *i*) ;;\n\
          *) return;;\n\
    esac\n\
\n\
    HISTCONTROL=ignoreboth\n\
\n\
    shopt -s histappend\n\
\n\
    HISTSIZE=1000\n\
    HISTFILESIZE=2000\n\
\n\
    shopt -s checkwinsize\n\
\n\
\n\
    [ -x /usr/bin/lesspipe ] && eval \"\$(SHELL=/bin/sh lesspipe)\"\n\
\n\
    if [ -z \"\${debian_chroot:-}\" ] && [ -r /etc/debian_chroot ]; then\n\
        debian_chroot=\$(cat /etc/debian_chroot)\n\
    fi\n\
\n\
    case \"\$TERM\" in\n\
        xterm-color|*-256color) color_prompt=yes;;\n\
    esac\n\
\n\
\n\
    if [ -n \"\$force_color_prompt\" ]; then\n\
        if [ -x /usr/bin/tput ] && tput setaf 1 >&/dev/null; then\n\
        color_prompt=yes\n\
        else\n\
        color_prompt=\n\
        fi\n\
    fi\n\
\n\
    if [ \"\$color_prompt\" = yes ]; then\n\
        PS1='\${debian_chroot:+(\$debian_chroot)}\[\033[07;32m\]\n[\d \t] \u@\h\[\033[00m\]:\[\033[07;34m\]\w\[\033[00m\]\n\\$ '\n\
    else\n\
        PS1='\${debian_chroot:+(\$debian_chroot)}\u@\h:\w\\$ '\n\
    fi\n\
    unset color_prompt force_color_prompt\n\
\n\
    case \"\$TERM\" in\n\
    xterm*|rxvt*)\n\
        PS1=\"\[\e]0;\${debian_chroot:+(\$debian_chroot)}\u@\h: \w\a\]\$PS1\"\n\
        ;;\n\
    *)\n\
        ;;\n\
    esac\n\
\n\
    if [ -x /usr/bin/dircolors ]; then\n\
        test -r ~/.dircolors && eval \"\$(dircolors -b ~/.dircolors)\" || eval \"\$(dircolors -b)\"\n\
        alias ls='ls --color=auto'\n\
\n\
        alias grep='grep --color=auto'\n\
        alias fgrep='fgrep --color=auto'\n\
        alias egrep='egrep --color=auto'\n\
    fi\n\
\n\
\n\
    alias ll='ls -alF'\n\
    alias la='ls -A'\n\
    alias l='ls -CF'\n\
\n\
    alias alert='notify-send --urgency=low -i \"\$([ \$? = 0 ] && echo terminal || echo error)\" \"\$(history|tail -n1|sed -e '\''s/^\s*[0-9]\+\s*//;s/[;&|]\s*alert\$//'\'')\"'\n\
\n\
\n\
    if [ -f ~/.bash_aliases ]; then\n\
        . ~/.bash_aliases\n\
    fi\n\
\n\
    if ! shopt -oq posix; then\n\
      if [ -f /usr/share/bash-completion/bash_completion ]; then\n\
        . /usr/share/bash-completion/bash_completion\n\
      elif [ -f /etc/bash_completion ]; then\n\
        . /etc/bash_completion\n\
      fi\n\
    fi" > /home/docker/.bashrc


EXPOSE 22
ENTRYPOINT ["/root/start-sshd.sh"]
