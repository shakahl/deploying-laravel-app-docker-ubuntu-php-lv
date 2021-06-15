# Deploying Laravel Aapp Ubuntu Php LV Docker

Used to create base image for running the laravel app

https://github.com/haakco/deploying-laravel-app

## Description

We are going to start by creating a base image for PHP.

This image will hold everything required except the Laravel code.

The image will also have NGINX built in to make our lives simpler.

To make logging simpler we'll try to send all logs to syslog. We'll then use syslog to send to stdout and stderr.

This makes it simpler to get logs.

You would then either look at the running container logs or pipe these logs to something like the ELK stack.

This image doesn't contain any Laravel code.

This is to save time rebuilding the whole image every time we do a code change.

Bellow is the top of our Docker file where we set these.

## Base PHP Docker Image

```dockerfile
ARG BASE_UBUNTU_VERSION='ubuntu:20.04'

FROM ${BASE_UBUNTU_VERSION}

ARG BASE_UBUNTU_VERSION='ubuntu:20.04'
ARG PHP_VERSION='7.4'

ENV DEBIAN_FRONTEND="noninteractive" \
    LANG="en_US.UTF-8" \
    LANGUAGE="en_US.UTF-8" \
    LC_ALL="C.UTF-8" \
    TERM="xterm" \
    PHP_VERSION="$PHP_VERSION"

RUN echo "PHP_VERSION=${PHP_VERSION}" && \
    echo "UBUNTU_VERSION=${BASE_UBUNTU_VERSION}" && \
    echo ""
```

To make future upgrading easier, there are variables for PHP and Ubuntu versions.

After this, we follow a similar process to the previous stages.

I'm mainly following the installation script we used in Stage 1. (Remember how I said you'd be re-using this)

For reference the installation script is here [https://github.com/haakco/deploying-laravel-app-stage1-simple-deploy/blob/main/setupCommands.sh](https://github.com/haakco/deploying-laravel-app-stage1-simple-deploy/blob/main/setupCommands.sh).

The one exception is we don't have to generate the SSL certificate, as we'll do that with a proxy that we'll run in
front of the server.

We'll set some flags to speed up the apt install.

First make sure the Ubuntu is entirely up to date.

We also install some packages to make our lives easier if we want to test anything. e.g., Ping and database clients.

[SupervisorD](http://supervisord.org/) is added  to run our services on start. It will also handle
restarting them if they crash.

You'll see after each run command, we do a cleanup to keep each layer as small as possible.

```dockerfile
## Setting to improve build speed
RUN echo 'debconf debconf/frontend select Noninteractive' | debconf-set-selections && \
    echo apt-fast apt-fast/maxdownloads string 10 | debconf-set-selections && \
    echo apt-fast apt-fast/dlflag boolean true | debconf-set-selections && \
    echo apt-fast apt-fast/aptmanager string apt-get | debconf-set-selections && \
    echo "force-unsafe-io" > /etc/dpkg/dpkg.cfg.d/02apt-speedup && \
    echo "Acquire::http {No-Cache=True;};" > /etc/apt/apt.conf.d/no-cache

## Make sure everything is fully upgraded and shared tooling
RUN apt update && \
    apt -y  \
        dist-upgrade \
        && \
    apt-get install -qy \
        bash-completion \
        ca-certificates \
        inetutils-ping inetutils-tools \
        logrotate \
        mysql-client \
        postgresql-client \
        rsyslog \
        software-properties-common sudo supervisor \
        vim \
        && \
    update-ca-certificates --fresh && \
    apt-get -y autoremove && \
    apt-get -y clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /var/tmp/* && \
    rm -rf /tmp/*
```

Next, we want to install PHP. We also install xdebug but disable it. The config for xdebug is updated to allow for
remote debugging.

When needed xdebug can be enabled. This is handled via environmental variables in the run script.

You'll see the PHP_VERSION variable previously setup up.

```dockerfile
## Install PHP disable xdebug
RUN add-apt-repository -y ppa:ondrej/php && \
    apt update && \
    apt install -y php${PHP_VERSION} php${PHP_VERSION}-cli php${PHP_VERSION}-fpm \
      php${PHP_VERSION}-bcmath \
      php${PHP_VERSION}-common php${PHP_VERSION}-curl \
      php${PHP_VERSION}-dev \
      php${PHP_VERSION}-gd php${PHP_VERSION}-gmp php${PHP_VERSION}-grpc \
      php${PHP_VERSION}-igbinary php${PHP_VERSION}-imagick php${PHP_VERSION}-intl \
      php${PHP_VERSION}-mcrypt php${PHP_VERSION}-mbstring php${PHP_VERSION}-mysql \
      php${PHP_VERSION}-opcache \
      php${PHP_VERSION}-pcov php${PHP_VERSION}-pgsql php${PHP_VERSION}-protobuf \
      php${PHP_VERSION}-redis \
      php${PHP_VERSION}-soap php${PHP_VERSION}-sqlite3 php${PHP_VERSION}-ssh2  \
      php${PHP_VERSION}-xml php${PHP_VERSION}-xdebug \
      php${PHP_VERSION}-zip \
      && \
    apt -y  \
        dist-upgrade \
        && \
    phpdismod -v ${PHP_VERSION} xdebug && \
    apt-get -y autoremove && \
    apt-get -y clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /var/tmp/* && \
    rm -rf /tmp/*

ADD ./files/php-modules/xdebug.ini /etc/php/${PHP_VERSION}/mods-available/xdebug.ini
ADD ./files/php-modules/igbinary.ini /etc/php/${PHP_VERSION}/mods-available/igbinary.ini
```

Now let us install Nginx and remove the default site.

```dockerfile
## Install nginx
RUN add-apt-repository -y ppa:nginx/stable && \
    apt update && \
    apt install -y \
        nginx \
      && \
    apt -y  \
        dist-upgrade \
        && \
    apt-get -y autoremove && \
    apt-get -y clean && \
    rm -rf /etc/nginx/sites-enabled/default && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /var/tmp/* && \
    rm -rf /tmp/*
```

To configure Nginx the way we want we'll copy the config files we want into the image.

For this, we'll create a ```files``` subdirectory directory and add the config files for Nginx in a subdirectory.

[https://github.com/haakco/deploying-laravel-app-docker-ubuntu-php-lv/tree/main/files/nginx_config](https://github.com/haakco/deploying-laravel-app-docker-ubuntu-php-lv/tree/main/files/nginx_config)

We also make sure that the Nginx is not running in daemon mode. This lets SupervisorD control when it starts or stops.

Please go over the config files to see how things are set up.

The config files are copied into the image during the build.

```dockerfile
ADD ./files/nginx_config /site/nginx/config
```

Next, we want to install the composer.

```dockerfile
RUN php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');" && \
    php composer-setup.php --install-dir=/bin --filename=composer && \
    php -r "unlink('composer-setup.php');"
```

To allow for some simpler debugging, we are going to add ssh. This allows using tools like Tinkerwell for simpler debugging.

```dockerfile
# Add openssh
RUN apt-get update && \
    apt-get -qy dist-upgrade && \
    apt-get install -qy \
      openssh-server \
      && \
    ssh-keygen -A && \
    mkdir -p /run/sshd && \
    apt-get -y autoremove && \
    apt-get -y clean && \
    rm -rf /var/lib/apt/lists/* && \
    rm -rf /var/tmp/* && \
    rm -rf /tmp/*
```

Next we want to have the ability to become the www-data user. So we edit the ```/etc/passwd```.

```dockerfile
## Allow log in as user
RUN sed -i.bak -E \
      -s 's#/var/www:/usr/sbin/nologin#/var/www:/bin/bash#' \
      /etc/passwd
```

While doing dev or debugging it would be nice to have tab completion for artisan and composer.

We also want to easily run the installed composer programs without having to put the full path in.

Finally, we improve some the history settings.

```dockerfile
## Add tab completion
ADD ./files/bash/artisan-bash-prompt /etc/bash_completion.d/artisan-bash-prompt
ADD ./files/bash/composer-bash-prompt /etc/bash_completion.d/composer-bash-prompt

# Set up bash variables
RUN echo 'PATH="/usr/bin:/var/www/site/vendor/bin:/var/www/site/vendor/bin:/site/.composer/vendor/bin:${PATH}"' >> /var/www/.bashrc && \
    echo 'PATH="/usr/bin:/var/www/site/vendor/bin:/var/www/site/vendor/bin:/site/.composer/vendor/bin:${PATH}"' >> /root/.bashrc && \
    echo 'shopt -s histappend' >> /var/www/.bashrc && \
    echo 'shopt -s histappend' >> /root/.bashrc && \
    echo 'PROMPT_COMMAND="history -a;$PROMPT_COMMAND"' >> /var/www/.bashrc && \
    echo 'PROMPT_COMMAND="history -a;$PROMPT_COMMAND"' >> /root/.bashrc && \
    echo 'cd /var/www/site' >> /var/www/.bashrc && \
    echo 'cd /var/www/site' >> /root/.bashrc && \
    touch /root/.bash_profile /var/www/.bash_profile && \
    chown root: /etc/bash_completion.d/artisan-bash-prompt /etc/bash_completion.d/composer-bash-prompt && \
    chmod u+rw /etc/bash_completion.d/artisan-bash-prompt /etc/bash_completion.d/composer-bash-prompt && \
    chmod go+r /etc/bash_completion.d/artisan-bash-prompt /etc/bash_completion.d/composer-bash-prompt && \
    mkdir -p /var/www/site/tmp
```

Next we add a configuration for log rotate. This prevents any log files in the running container from growing to large.

We also add a scrip that will pass all the environmental variable to any scripts we want to run.

```dockerfile
ADD ./files/logrotate.d/ /etc/logrotate.d/
ADD ./files/run_with_env.sh /bin/run_with_env.sh
```

To make the resulting image file more flexible we use environmental variables to change some settings in
the ```start.sh``` script.
(I'll cover the  ```start.sh``` once we've covered everything in the Dockerfile)

We set the default values for these next.

```dockerfile
ENV CRONTAB_ACTIVE="FALSE" \
    ENABLE_DEBUG="FALSE" \
    INITIALISE_FILE="/var/www/initialise.sh" \
    GEN_LV_ENV="FALSE" \
    LV_DO_CACHING="FALSE" \
    ENABLE_HORIZON="FALSE" \
    ENABLE_SIMPLE_QUEUE="FALSE" \
    SIMPLE_WORKER_NUM="5" \
    ENABLE_SSH="FALSE"

ENV PHP_TIMEZONE="UTC" \
    PHP_UPLOAD_MAX_FILESIZE="128M" \
    PHP_POST_MAX_SIZE="128M" \
    PHP_MEMORY_LIMIT="1G" \
    PHP_MAX_EXECUTION_TIME="60" \
    PHP_MAX_INPUT_TIME="60" \
    PHP_DEFAULT_SOCKET_TIMEOUT="60" \
    PHP_OPCACHE_MEMORY_CONSUMPTION="128" \
    PHP_OPCACHE_INTERNED_STRINGS_BUFFER="16" \
    PHP_OPCACHE_MAX_ACCELERATED_FILES="16229" \
    PHP_OPCACHE_REVALIDATE_PATH="1" \
    PHP_OPCACHE_ENABLE_FILE_OVERRIDE="0" \
    PHP_OPCACHE_VALIDATE_TIMESTAMPS="0" \
    PHP_OPCACHE_REVALIDATE_FREQ="1"

ENV PHP_OPCACHE_PRELOAD_FILE="" \
    COMPOSER_PROCESS_TIMEOUT=2000
```

We add the startup script we've mentioned previously (I'll cover what it does further down).

```dockerfile
# Script that are used to setup the container
ADD ./files/initialise.sh /var/www/initialise.sh
ADD ./files/start.sh /start.sh
ADD ./files/supervisord_base.conf /supervisord_base.conf

RUN chown www-data: -R /var/www/initialise.sh /var/www && \
    chmod a+x /var/www/initialise.sh && \
    chmod a+x /start.sh
```

Next we make sure that the files have the correct ownership.

```dockerfile
## Make sure directories and stdout and stderro have correct rights
RUN chmod -R a+w /dev/stdout && \
    chmod -R a+w /dev/stderr && \
    chmod -R a+w /dev/stdin && \
    usermod -a -G tty syslog && \
    usermod -a -G tty www-data && \
    find /var/www -not -user www-data -execdir chown "www-data" {} \+
```

We then also set up the health test and set the start script to be the default thing run.

```dockerfile
WORKDIR /var/www/site

ADD ./files/healthCheck.sh /healthCheck.sh

RUN chown www-data: /healthCheck.sh && \
    chmod a+x /healthCheck.sh

HEALTHCHECK \
  --interval=30s \
  --timeout=30s \
  --start-period=15s \
  --retries=10 \
  CMD /healthCheck.sh

CMD ["/start.sh"]
```

That covers everything in the Docker file.

## Base PHP Docker Startup Scripts

Let's just go over what
the [https://github.com/haakco/deploying-laravel-app-docker-ubuntu-php-lv/blob/main/files/start.sh](https://github.com/haakco/deploying-laravel-app-docker-ubuntu-php-lv/blob/main/files/start.sh)
does.

This file gets run every time we start the image and is used to configure the running container.

First we make sure that certain directories we need exist.

```shell
mkdir -p /var/www/site
mkdir -p /var/log/supervisor
mkdir -p /run/php
```

Next we set up some default environmental variables. You'll see these are a repeat of the ones in our Dockerfile.

You'll also see that if they any previously set enviroment variable values take preference.

This allows us to change how the container is configured and what will run by just altering the variables.

```shell
## All the following setting can be overwritten by passing environmental variables on the docker run
export PHP_VERSION=${PHP_VERSION:-7.4}

export CRONTAB_ACTIVE=${CRONTAB_ACTIVE:-FALSE}
export ENABLE_DEBUG=${ENABLE_DEBUG:-FALSE}

export INITIALISE_FILE=${INITIALISE_FILE:-'/var/www'}

export GEN_LV_ENV=${GEN_LV_ENV:-FALSE}
export LV_DO_CACHING=${LV_DO_CACHING:-FALSE}

export ENABLE_HORIZON=${ENABLE_HORIZON:-FALSE}
export ENABLE_SIMPLE_QUEUE=${ENABLE_SIMPLE_QUEUE:-FALSE}
export SIMPLE_WORKER_NUM=${SIMPLE_WORKER_NUM:-5}

export ENABLE_SSH=${ENABLE_SSH:-FALSE}

export PHP_TIMEZONE=${PHP_TIMEZONE:-"UTC"}
export PHP_UPLOAD_MAX_FILESIZE=${PHP_UPLOAD_MAX_FILESIZE:-"128M"}
export PHP_POST_MAX_SIZE=${PHP_POST_MAX_SIZE:-"128M"}
export PHP_MEMORY_LIMIT=${PHP_MEMORY_LIMIT:-"1G"}
export PHP_MAX_EXECUTION_TIME=${PHP_MAX_EXECUTION_TIME:-"60"}
export PHP_MAX_INPUT_TIME=${PHP_MAX_INPUT_TIME:-"60"}
export PHP_DEFAULT_SOCKET_TIMEOUT=${PHP_DEFAULT_SOCKET_TIMEOUT:-"60"}
export PHP_OPCACHE_MEMORY_CONSUMPTION=${PHP_OPCACHE_MEMORY_CONSUMPTION:-"128"}
export PHP_OPCACHE_INTERNED_STRINGS_BUFFER=${PHP_OPCACHE_INTERNED_STRINGS_BUFFER:-"16"}
export PHP_OPCACHE_MAX_ACCELERATED_FILES=${PHP_OPCACHE_MAX_ACCELERATED_FILES:-"16229"}
export PHP_OPCACHE_REVALIDATE_PATH=${PHP_OPCACHE_REVALIDATE_PATH:-"1"}
export PHP_OPCACHE_ENABLE_FILE_OVERRIDE=${PHP_OPCACHE_ENABLE_FILE_OVERRIDE:-"0"}
export PHP_OPCACHE_VALIDATE_TIMESTAMPS=${PHP_OPCACHE_VALIDATE_TIMESTAMPS:-"0"}
export PHP_OPCACHE_REVALIDATE_FREQ=${PHP_OPCACHE_REVALIDATE_FREQ:-"1"}
export PHP_OPCACHE_PRELOAD_FILE=${PHP_OPCACHE_PRELOAD_FILE:-""}

export COMPOSER_PROCESS_TIMEOUT=${COMPOSER_PROCESS_TIMEOUT:-2000}
```

The php.ini is altered using the environmental variables above.

```shell
sed -i \
  -e "s@date.timezone =.*@date.timezone = ${PHP_TIMEZONE}@" \
  -e "s/upload_max_filesize = .*/upload_max_filesize = ${PHP_UPLOAD_MAX_FILESIZE}/" \
  -e "s/post_max_size = .*/post_max_size = ${PHP_POST_MAX_SIZE}/"  \
  -e "s/memory_limit = .*/memory_limit = ${PHP_MEMORY_LIMIT}/" \
  -e "s/max_execution_time = .*/max_execution_time = ${PHP_MAX_EXECUTION_TIME}/" \
  -e "s/max_input_time = .*/max_input_time = ${PHP_MAX_INPUT_TIME}/" \
  -e "s/default_socket_timeout = .*/default_socket_timeout = ${PHP_DEFAULT_SOCKET_TIMEOUT}/" \
  -e "s/opcache.memory_consumption=.*/opcache.memory_consumption=${PHP_OPCACHE_MEMORY_CONSUMPTION}/" \
  -e "s/opcache.interned_strings_buffer=.*/opcache.interned_strings_buffer=${PHP_OPCACHE_INTERNED_STRINGS_BUFFER}/" \
  -e "s/.*opcache.max_accelerated_files=.*/opcache.max_accelerated_files=${PHP_OPCACHE_MAX_ACCELERATED_FILES}/" \
  -e "s/opcache.revalidate_path=.*/opcache.revalidate_path=${PHP_OPCACHE_REVALIDATE_PATH}/" \
  -e "s/opcache.enable_file_override=.*/opcache.enable_file_override=${PHP_OPCACHE_ENABLE_FILE_OVERRIDE}/" \
  -e "s/opcache.validate_timestamps=.*/opcache.validate_timestamps=${PHP_OPCACHE_VALIDATE_TIMESTAMPS}/" \
  -e "s/opcache.revalidate_freq=.*/opcache.revalidate_freq=${PHP_OPCACHE_REVALIDATE_FREQ}/" \
  /etc/php/"${PHP_VERSION}"/cli/php.ini \
  /etc/php/"${PHP_VERSION}"/fpm/php.ini

sed -Ei \
  -e "s/error_log = .*/error_log = syslog/" \
  -e "s/.*syslog\.ident = .*/syslog.ident = php-fpm/" \
  -e "s/.*log_buffering = .*/log_buffering = yes/" \
  /etc/php/${PHP_VERSION}/fpm/php-fpm.conf
echo "request_terminate_timeout = 600" >> /etc/php/${PHP_VERSION}/fpm/php-fpm.conf

sed -Ei \
  -e "s/^user = .*/user = www-data/" \
  -e "s/^group = .*/group = www-data/" \
  -e 's/listen\.owner.*/listen.owner = www-data/' \
  -e 's/listen\.group.*/listen.group = www-data/' \
  -e 's/.*listen\.backlog.*/listen.backlog = 65536/' \
  -e "s/pm\.max_children = .*/pm.max_children = 32/" \
  -e "s/pm\.start_servers = .*/pm.start_servers = 4/" \
  -e "s/pm\.min_spare_servers = .*/pm.min_spare_servers = 4/" \
  -e "s/pm\.max_spare_servers = .*/pm.max_spare_servers = 16/" \
  -e "s/.*pm\.max_requests = .*/pm.max_requests = 0/" \
  -e "s/.*pm\.status_path = .*/pm.status_path = \/fpm-status/" \
  -e "s/.*ping\.path = .*/ping.path = \/fpm-ping/" \
  -e 's/\/run\/php\/.*fpm.sock/\/run\/php\/fpm.sock/' \
  /etc/php/${PHP_VERSION}/fpm/pool.d/www.conf
```

Next if you would like to do some opcache preloading you set a file to do this.

https://stitcher.io/blog/preloading-in-php-74

```shell
if [[ "${PHP_OPCACHE_PRELOAD_FILE}" != "" ]]; then
  sed -i \
    -e "s#;opcache.preload=.*#opcache.preload=${PHP_OPCACHE_PRELOAD_FILE}#" \
    -e "s#;opcache.preload_user=.*#opcache.preload_user=www-data#" \
    /etc/php/"${PHP_VERSION}"/fpm/php.ini
fi
```

Next we set up supervisor config and what should run.

As part of this you can choose to use Horizon or the simple queue worker.

I would recommend rather using Horizon as it has several advantages.

```shell
cp /supervisord_base.conf /supervisord.conf

if [[ "${ENABLE_HORIZON}" = "TRUE" ]]; then
  sed -E -i -e 's/^numprocs=ENABLE_HORIZON/numprocs=1/' /supervisord.conf
  SIMPLE_WORKER_NUM='0'
  ENABLE_SIMPLE_QUEUE='FALSE'
else
  sed -E -i -e 's/^numprocs=ENABLE_HORIZON/numprocs=0/' /supervisord.conf
fi

sed -E -i -e 's/^numprocs=WORKER_NUM/numprocs='"${WORKERS}"'/' /supervisord.conf

if [[ "${ENABLE_HORIZON}" != "TRUE" && "${ENABLE_SIMPLE_QUEUE}" = "TRUE" ]]; then
  sed -E -i -e 's/SIMPLE_WORKER_NUM/'"${SIMPLE_WORKER_NUM}"'/' /supervisord.conf
else
  sed -E -i -e 's/SIMPLE_WORKER_NUM/0/' /supervisord.conf
fi

if [[ "${ENABLE_SSH}" = "TRUE" ]]; then
  sed -E -i -e 's/ENABLE_SSH/1/' /supervisord.conf
else
  sed -E -i -e 's/ENABLE_SSH/0/' /supervisord.conf
fi

sed -E -i -e "s/PHP_VERSION/${PHP_VERSION}/g" /supervisord.conf
```

When enabling the ssh server there is also the option to add your ssh key via an environmental variable.

```shell
mkdir -p /root/.ssh/
chmod 700 /root/.ssh
ssh-keyscan github.com >> /root/.ssh/known_hosts
chmod 600 /root/.ssh/
touch /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys

if [[ ! -z "${SSH_AUTHORIZED_KEYS}" ]];then
  echo "${SSH_AUTHORIZED_KEYS}" > /root/.ssh/authorized_keys
fi

chmod 600 /root/.ssh/authorized_keys
```

Next we set up the crontab.

By default, you'll se we have logrotate and a chown on start just in case.

We then have a variable, so you can decide if you want to enable Laravel's task scheduling.

https://laravel.com/docs/master/scheduling

```shell
cat > ${TEMP_CRON_FILE} <<- EndOfMessage
# m h  dom mon dow   command
0 * * * * /usr/sbin/logrotate -vf /etc/logrotate.d/*.auto 2>&1 | /dev/stdout

#rename on start
@reboot find /var/www -not -user www-data -execdir chown "www-data:" {} \+ | /dev/stdout

EndOfMessage

if [[ "${CRONTAB_ACTIVE}" = "TRUE" ]]; then
 cat >> ${TEMP_CRON_FILE} <<- EndOfMessage
* * * * * su www-data -c '/usr/bin/php /var/www/site/artisan schedule:run' 2>&1 >> /var/log/cron.log
EndOfMessage
fi

cat ${TEMP_CRON_FILE} | crontab -

rm ${TEMP_CRON_FILE}
```

Next is where can choose to enable xdebug.

```shell
if [[ "${ENABLE_DEBUG}" = "TRUE" ]]; then
  phpenmod -v "${PHP_VERSION}" xdebug
fi
```

By default, you can set the environmental variables for laravel directly.

Or if you would like to generate a ```.env``` file you can pass them prefixed with ```LVENV_``` and set ```GEN_LV_ENV```
to ```TRUE```.

The main advantage is it makes it simpler to see what the settings are by looking at the ```.env``` file.

```shell
if [[ "${GEN_LV_ENV}" = "TRUE" ]]; then
  env | grep 'LVENV_' | sort | sed -E -e 's/"/\\"/g' -e 's#LVENV_(.*)=#\1=#' -e 's#=(.+)#="\1"#' > /var/www/site/.env
fi
```

Next we override composers timeout. This is need if you are far from the composer servers. e.g. in Africa.

```shell
composer config --global process-timeout "${COMPOSER_PROCESS_TIMEOUT}"
```

Next we make sure the ```stdout``` add ```stdin``` are accessible.

```shell
# Try to fix rsyslogd: file '/dev/stdout': open error: Permission denied
chmod -R a+w /dev/stdout
chmod -R a+w /dev/stderr
chmod -R a+w /dev/stdin
```

We allow you to pass your own initialise file. This let you create a script that will do what ever setup you want before the server is up.

It generally will have things like composer install.

```shell
if [[ -e "${INITIALISE_FILE}" ]]; then
  chown www-data: "${INITIALISE_FILE}"
  chmod u+x "${INITIALISE_FILE}"
  mkdir /root/.composer /var/www/.composer
  chmod a+r /root/.composer /var/www/.composer
  su www-data --preserve-environment -c "${INITIALISE_FILE}" >> /var/log/initialise.log
fi
```

Finally, we do a quick logrotate just in case and start supervisor.

```shell
## Rotate logs at start just in case
/usr/sbin/logrotate -vf /etc/logrotate.d/*.auto &

/usr/bin/supervisord -n -c /supervisord.conf
```

### Simple local dev enviroment
Ok in this step we are going to set up a local dev enviroment using the php image we created above.

We'll also spin up a Redis and MySQL. Though for those we'll use the default images on [docker hub](https://hub.docker.com/).

I'm going to first show you how to do this via the command line.

I'll then give you docker-compose files to do this.

The compose files are slightly easier to ready for people who are not used to command line.
