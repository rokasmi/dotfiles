ARG PHP_VERSION=${PHP_VERSION}

FROM letsdockerize/laradock-workspace:2.3-${PHP_VERSION}

# Start as root
USER root

###########################################################################
# Laradock non-root user:
###########################################################################

# Add a non-root user to prevent files being created with root permissions on host machine.
ARG PUID=1000
ENV PUID ${PUID}
ARG PGID=1000
ENV PGID ${PGID}

ARG PHP_VERSION=${PHP_VERSION}
ARG PHP_MEMORY_LIMIT=${PHP_MEMORY_LIMIT}

COPY ./keyboard /etc/default/keyboard

# always run apt update when start and after add new source list, then clean up at end.
RUN apt-get update -yqq && \
    apt-get install -yqq graphviz curl wget zip unzip zsh vim ruby-full python-pip mysql-client mlocate net-tools xclip sudo libssl-dev safe-rm && \
#    apt-get install -yqq php-imagick php-apcu php${PHP_VERSION}-soap php${PHP_VERSION}-sybase php${PHP_VERSION}-fpm && \
    gem update --system && gem install bundler sass && \
    pecl channel-update pecl.php.net && \
    pip install Pygments

RUN groupadd -g ${PGID} laradock
RUN useradd -u ${PUID} -g laradock -m laradock -G docker_env
RUN usermod -p "*" laradock

COPY ./freetds.conf /etc/freetds/freetds.conf
COPY ./php-cli.ini /etc/php/${PHP_VERSION}/cli/php.ini
COPY ./php-fpm.ini /etc/php/${PHP_VERSION}/fpm/php.ini

RUN composer self-update --stable

###########################################################################
# Set Timezone
###########################################################################

ARG TZ=UTC
ENV TZ ${TZ}

RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

###########################################################################
# Oh my zsh shell
###########################################################################

USER laradock

RUN wget https://github.com/robbyrussell/oh-my-zsh/raw/master/tools/install.sh -O - | zsh || true
COPY ./.zshrc /home/laradock/.zshrc
COPY ./.dir_colors /home/laradock/.dir_colors
ENV ZSH_CUSTOM=/home/laradock/.oh-my-zsh/custom
RUN git clone git://github.com/zsh-users/zsh-autosuggestions $ZSH_CUSTOM/plugins/zsh-autosuggestions

USER root

# Shared directory
RUN chown laradock:laradock /home/laradock/.zshrc
RUN mkdir -p /home/laradock/shared
RUN chown -R laradock:laradock /home/laradock/shared

###########################################################################
# User Aliases
###########################################################################

USER root

COPY ./aliases.sh /root/aliases.sh
COPY ./aliases.sh /home/laradock/aliases.sh

RUN sed -i 's/\r//' /root/aliases.sh && \
    sed -i 's/\r//' /home/laradock/aliases.sh && \
    chown laradock:laradock /home/laradock/aliases.sh && \
    echo "" >> ~/.bashrc && \
    echo "# Load Custom Aliases" >> ~/.bashrc && \
    echo "source ~/aliases.sh" >> ~/.bashrc && \
	echo "" >> ~/.bashrc

RUN apt-get install -yq zsh

USER laradock

RUN echo "" >> ~/.bashrc && \
    echo "# Load Custom Aliases" >> ~/.bashrc && \
    echo "source ~/aliases.sh" >> ~/.bashrc && \
	echo "" >> ~/.bashrc

###########################################################################
# Symfony installer:
###########################################################################

USER laradock

RUN wget https://get.symfony.com/cli/installer -O - | bash
# Export composer vendor path
RUN echo "" >> ~/.bashrc && \
    echo 'export PATH="~/.symfony/bin:$PATH"' >> ~/.bashrc

###########################################################################
# Composer:
###########################################################################

USER root

# Add the composer.json
COPY ./composer.json /home/laradock/.composer/composer.json

# Make sure that ~/.composer belongs to laradock
RUN chown -R laradock:laradock /home/laradock/.composer

USER laradock

# Export composer vendor path
RUN echo "" >> ~/.bashrc && \
    echo 'export PATH="~/.composer/vendor/bin:$PATH"' >> ~/.bashrc

RUN composer global require hirak/prestissimo

###########################################################################
# MongoDB:
###########################################################################

#USER root
#ARG PHP_VERSION=${PHP_VERSION}
#
## Install the mongodb extension
#RUN pecl -q install mongodb && \
#    echo "extension=mongodb.so" >> /etc/php/${PHP_VERSION}/mods-available/mongodb.ini && \
#    ln -s /etc/php/${PHP_VERSION}/mods-available/mongodb.ini /etc/php/${PHP_VERSION}/cli/conf.d/30-mongodb.ini

###########################################################################
# PHP REDIS EXTENSION
###########################################################################

#USER root
#ARG PHP_VERSION=${PHP_VERSION}
#
## Install Php Redis extension
#RUN printf "\n" | pecl -q install -o -f redis && \
#    echo "extension=redis.so" >> /etc/php/${PHP_VERSION}/mods-available/redis.ini && \
#    phpenmod redis

###########################################################################
# xDebug:
###########################################################################

#USER root
#ARG PHP_VERSION=${PHP_VERSION}
#
#RUN apt-get install -y php${PHP_VERSION}-xdebug && \
#    sed -i 's/^;//g' /etc/php/${PHP_VERSION}/cli/conf.d/20-xdebug.ini && \
#    sed -i 's/^;//g' /etc/php/${PHP_VERSION}/fpm/conf.d/20-xdebug.ini
#
#USER root
#ARG PHP_VERSION=${PHP_VERSION}
#
## Copy xdebug configuration for remote debugging
#COPY ./xdebug.ini /etc/php/${PHP_VERSION}/cli/conf.d/xdebug.ini
#COPY ./xdebug.ini /etc/php/${PHP_VERSION}/fpm/conf.d/xdebug.ini

###########################################################################
# Node / NVM:
###########################################################################

USER laradock

# Check if NVM needs to be installed
ARG INSTALL_NODE=true
ARG NPM_REGISTRY
ENV NPM_REGISTRY ${NPM_REGISTRY}
ENV NVM_DIR /home/laradock/.nvm

RUN mkdir -p /home/laradock/.nvm
RUN if [ ${INSTALL_NODE} = true ]; then \
    # Install nvm (A Node Version Manager)
    curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.34.0/install.sh | bash && \
        . $NVM_DIR/nvm.sh && \
        nvm install --lts && \
        nvm use --lts && \
        if [ ${NPM_REGISTRY} ]; then \
        npm config set registry ${NPM_REGISTRY} \
        ;fi && \
        npm install -g bower gulp gulp-cli uglify-js uglifycss elasticdump && \
        ln -s `npm bin --global` /home/laradock/.node-bin \
;fi

# Wouldn't execute when added to the RUN statement in the above block
# Source NVM when loading bash since ~/.profile isn't loaded on non-login shell
RUN if [ ${INSTALL_NODE} = true ]; then \
    echo "" >> ~/.bashrc && \
    echo 'export NVM_DIR="$HOME/.nvm"' >> ~/.bashrc && \
    echo '[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm' >> ~/.bashrc \
;fi

# Add NVM binaries to root's .bashrc
USER root

RUN if [ ${INSTALL_NODE} = true ]; then \
    echo "" >> ~/.bashrc && \
    echo 'export NVM_DIR="/home/laradock/.nvm"' >> ~/.bashrc && \
    echo '[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm' >> ~/.bashrc \
;fi

# Add PATH for node
ENV PATH $PATH:/home/laradock/.node-bin

RUN if [ ${NPM_REGISTRY} ]; then \
    . ~/.bashrc && npm config set registry ${NPM_REGISTRY} \
;fi

###########################################################################
# YARN:
###########################################################################

USER laradock

ARG INSTALL_YARN=true
ARG YARN_VERSION=latest
ENV YARN_VERSION ${YARN_VERSION}

RUN if [ ${INSTALL_YARN} = true ]; then \
    [ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh" && \
    if [ ${YARN_VERSION} = "latest" ]; then \
        curl -o- -L https://yarnpkg.com/install.sh | bash; \
    else \
        curl -o- -L https://yarnpkg.com/install.sh | bash -s -- --version ${YARN_VERSION}; \
    fi && \
    echo "" >> ~/.bashrc && \
    echo 'export PATH="$HOME/.yarn/bin:$PATH"' >> ~/.bashrc \
;fi

# Add YARN binaries to root's .bashrc
USER root

RUN if [ ${INSTALL_YARN} = true ]; then \
    echo "" >> ~/.bashrc && \
    echo 'export YARN_DIR="/home/laradock/.yarn"' >> ~/.bashrc && \
    echo 'export PATH="$YARN_DIR/bin:$PATH"' >> ~/.bashrc \
;fi

###########################################################################
# wkhtmltopdf:
###########################################################################

RUN wget https://downloads.wkhtmltopdf.org/0.12/0.12.5/wkhtmltox_0.12.5-1.xenial_amd64.deb -P /tmp
RUN dpkg --configure -a
RUN apt-get install -y xvfb libfontconfig fontconfig libpng16-16 libxrender1 xfonts-75dpi build-essential xorg
RUN dpkg -i /tmp/wkhtmltox_0.12.5-1.xenial_amd64.deb
RUN apt-get update

#
#--------------------------------------------------------------------------
# Final Touch
#--------------------------------------------------------------------------
#

USER root

RUN echo "export PATH=$PATH" > /etc/environment

# Clean up
RUN apt-get clean && \
    rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/* && \
    rm /var/log/lastlog /var/log/faillog

# Set default work directory
WORKDIR /var/www

EXPOSE 8000-8005
