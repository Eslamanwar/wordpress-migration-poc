# Migrating Legacy app from Virtualization to containerization POC

## Description
- This is show case how to migrate lagacy application based on wordpress hosted on VM to kubernetes container

### The legacy system is :
- MYSQL Database VM --> hosted on VM 192.168.0.40
- wordpress Application --> hosted on VM (192.168.0.44)
    -- here wordpress app is 2 parts (PHP wordpress and Apache web-server)
- Nginx acts as reverse proxy to route traffic to wordpress app --> hosted on VM (192.168.0.42)    



### what we intend to do here
- Keep the Mysql Database hosted on the same VM 192.168.0.40
- Containerize the wordpress app with the legacy app files (PHP files ,Apache web-server)
- Implement the Nginx-ingress container on k8s that will act as reverse proxy to the wordpress app




## Containerize the wordpress app with the legacy app files (PHP files ,Apache web-server)

- Copy the wordpress content from the legacy system (192.168.0.44)
```
cd /var/www
tar cvzf wp.tar.gz ./wordpress          ## compress the wordpress files and move the archive to the location that you will create the docker image
```



- create the docker image 
    - mkdir wordpress
    - cd wordpress 
    - tar -xvf wp.tar.gz                      ## extract the files where you will create the docker image
    - touch Dockerfile                        ## create file called Dockerfile 


```
FROM php:7.2-apache

# persistent dependencies
RUN set -eux; \
        apt-get update; \
        apt-get install -y --no-install-recommends \
# Ghostscript is required for rendering PDF previews
                ghostscript \
        ; \
        rm -rf /var/lib/apt/lists/*

# install the PHP extensions we need (https://make.wordpress.org/hosting/handbook/handbook/server-environment/#php-extensions)
RUN set -ex; \
        \
        savedAptMark="$(apt-mark showmanual)"; \
        \
        apt-get update; \
        apt-get install -y --no-install-recommends \
                libfreetype6-dev \
                libjpeg-dev \
                libmagickwand-dev \
                libpng-dev \
        ; \
        \
        docker-php-ext-configure gd --with-freetype-dir=/usr --with-jpeg-dir=/usr --with-png-dir=/usr; \
        docker-php-ext-install -j "$(nproc)" \
                bcmath \
                exif \
                gd \
                mysqli \
                opcache \
                zip \
        ; \
        pecl install imagick-3.4.4; \
        docker-php-ext-enable imagick; \
        \
# reset apt-mark's "manual" list so that "purge --auto-remove" will remove all build dependencies
        apt-mark auto '.*' > /dev/null; \
        apt-mark manual $savedAptMark; \
        ldd "$(php -r 'echo ini_get("extension_dir");')"/*.so \
                | awk '/=>/ { print $3 }' \
                | sort -u \
                | xargs -r dpkg-query -S \
                | cut -d: -f1 \
                | sort -u \
                | xargs -rt apt-mark manual; \
        \
        apt-get purge -y --auto-remove -o APT::AutoRemove::RecommendsImportant=false; \
        rm -rf /var/lib/apt/lists/*

# set recommended PHP.ini settings
# see https://secure.php.net/manual/en/opcache.installation.php
RUN { \
                echo 'opcache.memory_consumption=128'; \
                echo 'opcache.interned_strings_buffer=8'; \
                echo 'opcache.max_accelerated_files=4000'; \
                echo 'opcache.revalidate_freq=2'; \
                echo 'opcache.fast_shutdown=1'; \
        } > /usr/local/etc/php/conf.d/opcache-recommended.ini
# https://wordpress.org/support/article/editing-wp-config-php/#configure-error-logging
RUN { \
# https://www.php.net/manual/en/errorfunc.constants.php
# https://github.com/docker-library/wordpress/issues/420#issuecomment-517839670
                echo 'error_reporting = E_ERROR | E_WARNING | E_PARSE | E_CORE_ERROR | E_CORE_WARNING | E_COMPILE_ERROR | E_COMPILE_WARNING | E_RECOVERABLE_ERROR'; \
                echo 'display_errors = Off'; \
                echo 'display_startup_errors = Off'; \
                echo 'log_errors = On'; \
                echo 'error_log = /dev/stderr'; \
                echo 'log_errors_max_len = 1024'; \
                echo 'ignore_repeated_errors = On'; \
                echo 'ignore_repeated_source = Off'; \
                echo 'html_errors = Off'; \
        } > /usr/local/etc/php/conf.d/error-logging.ini

RUN set -eux; \
        a2enmod rewrite expires; \
        \
# https://httpd.apache.org/docs/2.4/mod/mod_remoteip.html
        a2enmod remoteip; \
        { \
                echo 'RemoteIPHeader X-Forwarded-For'; \
# these IP ranges are reserved for "private" use and should thus *usually* be safe inside Docker
                echo 'RemoteIPTrustedProxy 10.0.0.0/8'; \
                echo 'RemoteIPTrustedProxy 172.16.0.0/12'; \
                echo 'RemoteIPTrustedProxy 192.168.0.0/16'; \
                echo 'RemoteIPTrustedProxy 169.254.0.0/16'; \
                echo 'RemoteIPTrustedProxy 127.0.0.0/8'; \
        } > /etc/apache2/conf-available/remoteip.conf; \
        a2enconf remoteip; \
# https://github.com/docker-library/wordpress/issues/383#issuecomment-507886512
# (replace all instances of "%h" with "%a" in LogFormat)
        find /etc/apache2 -type f -name '*.conf' -exec sed -ri 's/([[:space:]]*LogFormat[[:space:]]+"[^"]*)%h([^"]*")/\1%a\2/g' '{}' +

COPY ./wordpress /var/www/html

VOLUME /var/www/html

ENV WORDPRESS_VERSION 5.3.1
ENV WORDPRESS_SHA1 3c635c9f6546782e0bb315784d4663d0e47f872e

RUN set -ex; \
        curl -o wordpress.tar.gz -fSL "https://wordpress.org/wordpress-${WORDPRESS_VERSION}.tar.gz"; \
        echo "$WORDPRESS_SHA1 *wordpress.tar.gz" | sha1sum -c -; \
# upstream tarballs include ./wordpress/ so this gives us /usr/src/wordpress
        tar -xzf wordpress.tar.gz -C /usr/src/; \
        rm wordpress.tar.gz; \
        chown -R www-data:www-data /usr/src/wordpress


COPY ./docker-entrypoint.sh /usr/local/bin/
RUN chmod 777  /usr/local/bin/docker-entrypoint.sh

ENTRYPOINT ["docker-entrypoint.sh"]
CMD ["apache2-foreground"]
```

- Be aware of this step in Dockerfile [COPY ./wordpress /var/www/html] where we copy the content of the legacy app



    - touch docker-entrypoint.sh            ## create this file as entrypoint to docker
```
#!/bin/bash
set -euo pipefail

# usage: file_env VAR [DEFAULT]
#    ie: file_env 'XYZ_DB_PASSWORD' 'example'
# (will allow for "$XYZ_DB_PASSWORD_FILE" to fill in the value of
#  "$XYZ_DB_PASSWORD" from a file, especially for Docker's secrets feature)
file_env() {
        local var="$1"
        local fileVar="${var}_FILE"
        local def="${2:-}"
        if [ "${!var:-}" ] && [ "${!fileVar:-}" ]; then
                echo >&2 "error: both $var and $fileVar are set (but are exclusive)"
                exit 1
        fi
        local val="$def"
        if [ "${!var:-}" ]; then
                val="${!var}"
        elif [ "${!fileVar:-}" ]; then
                val="$(< "${!fileVar}")"
        fi
        export "$var"="$val"
        unset "$fileVar"
}

if [[ "$1" == apache2* ]] || [ "$1" == php-fpm ]; then
        if [ "$(id -u)" = '0' ]; then
                case "$1" in
                        apache2*)
                                user="${APACHE_RUN_USER:-www-data}"
                                group="${APACHE_RUN_GROUP:-www-data}"

                                # strip off any '#' symbol ('#1000' is valid syntax for Apache)
                                pound='#'
                                user="${user#$pound}"
                                group="${group#$pound}"
                                ;;
                        *) # php-fpm
                                user='www-data'
                                group='www-data'
                                ;;
                esac
        else
                user="$(id -u)"
                group="$(id -g)"
        fi

        if [ ! -e index.php ] && [ ! -e wp-includes/version.php ]; then
                # if the directory exists and WordPress doesn't appear to be installed AND the permissions of it are root:root, let's chown it (likely a Docker-created directory)
                if [ "$(id -u)" = '0' ] && [ "$(stat -c '%u:%g' .)" = '0:0' ]; then
                        chown "$user:$group" .
                fi

                echo >&2 "WordPress not found in $PWD - copying now..."
                if [ -n "$(ls -A)" ]; then
                        echo >&2 "WARNING: $PWD is not empty! (copying anyhow)"
                fi
                sourceTarArgs=(
                        --create
                        --file -
                        --directory /usr/src/wordpress
                        --owner "$user" --group "$group"
                )
                targetTarArgs=(
                        --extract
                        --file -
                )
                if [ "$user" != '0' ]; then
                        # avoid "tar: .: Cannot utime: Operation not permitted" and "tar: .: Cannot change mode to rwxr-xr-x: Operation not permitted"
                        targetTarArgs+=( --no-overwrite-dir )
                fi
                tar "${sourceTarArgs[@]}" . | tar "${targetTarArgs[@]}"
                echo >&2 "Complete! WordPress has been successfully copied to $PWD"
                if [ ! -e .htaccess ]; then
                        # NOTE: The "Indexes" option is disabled in the php:apache base image
                        cat > .htaccess <<-'EOF'
                                # BEGIN WordPress
                                <IfModule mod_rewrite.c>
                                RewriteEngine On
                                RewriteBase /
                                RewriteRule ^index\.php$ - [L]
                                RewriteCond %{REQUEST_FILENAME} !-f
                                RewriteCond %{REQUEST_FILENAME} !-d
                                RewriteRule . /index.php [L]
                                </IfModule>
                                # END WordPress
                        EOF
                        chown "$user:$group" .htaccess
                fi
        fi

        # allow any of these "Authentication Unique Keys and Salts." to be specified via
        # environment variables with a "WORDPRESS_" prefix (ie, "WORDPRESS_AUTH_KEY")
        uniqueEnvs=(
                AUTH_KEY
                SECURE_AUTH_KEY
                LOGGED_IN_KEY
                NONCE_KEY
                AUTH_SALT
                SECURE_AUTH_SALT
                LOGGED_IN_SALT
                NONCE_SALT
        )
        envs=(
                WORDPRESS_DB_HOST
                WORDPRESS_DB_USER
                WORDPRESS_DB_PASSWORD
                WORDPRESS_DB_NAME
                WORDPRESS_DB_CHARSET
                WORDPRESS_DB_COLLATE
                "${uniqueEnvs[@]/#/WORDPRESS_}"
                WORDPRESS_TABLE_PREFIX
                WORDPRESS_DEBUG
                WORDPRESS_CONFIG_EXTRA
        )
        haveConfig=
        for e in "${envs[@]}"; do
                file_env "$e"
                if [ -z "$haveConfig" ] && [ -n "${!e}" ]; then
                        haveConfig=1
                fi
        done

        # linking backwards-compatibility
        if [ -n "${!MYSQL_ENV_MYSQL_*}" ]; then
                haveConfig=1
                # host defaults to "mysql" below if unspecified
                : "${WORDPRESS_DB_USER:=${MYSQL_ENV_MYSQL_USER:-root}}"
                if [ "$WORDPRESS_DB_USER" = 'root' ]; then
                        : "${WORDPRESS_DB_PASSWORD:=${MYSQL_ENV_MYSQL_ROOT_PASSWORD:-}}"
                else
                        : "${WORDPRESS_DB_PASSWORD:=${MYSQL_ENV_MYSQL_PASSWORD:-}}"
                fi
                : "${WORDPRESS_DB_NAME:=${MYSQL_ENV_MYSQL_DATABASE:-}}"
        fi

        # only touch "wp-config.php" if we have environment-supplied configuration values
        if [ "$haveConfig" ]; then
                : "${WORDPRESS_DB_HOST:=mysql}"
                : "${WORDPRESS_DB_USER:=root}"
                : "${WORDPRESS_DB_PASSWORD:=}"
                : "${WORDPRESS_DB_NAME:=wordpress}"
                : "${WORDPRESS_DB_CHARSET:=utf8}"
                : "${WORDPRESS_DB_COLLATE:=}"

                # version 4.4.1 decided to switch to windows line endings, that breaks our seds and awks
                # https://github.com/docker-library/wordpress/issues/116
                # https://github.com/WordPress/WordPress/commit/1acedc542fba2482bab88ec70d4bea4b997a92e4
                sed -ri -e 's/\r$//' wp-config*

                if [ ! -e wp-config.php ]; then
                        awk '
                                /^\/\*.*stop editing.*\*\/$/ && c == 0 {
                                        c = 1
                                        system("cat")
                                        if (ENVIRON["WORDPRESS_CONFIG_EXTRA"]) {
                                                print "// WORDPRESS_CONFIG_EXTRA"
                                                print ENVIRON["WORDPRESS_CONFIG_EXTRA"] "\n"
                                        }
                                }
                                { print }
                        ' wp-config-sample.php > wp-config.php <<'EOPHP'
// If we're behind a proxy server and using HTTPS, we need to alert WordPress of that fact
// see also http://codex.wordpress.org/Administration_Over_SSL#Using_a_Reverse_Proxy
if (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] === 'https') {
        $_SERVER['HTTPS'] = 'on';
}

EOPHP
                        chown "$user:$group" wp-config.php
                elif [ -e wp-config.php ] && [ -n "$WORDPRESS_CONFIG_EXTRA" ] && [[ "$(< wp-config.php)" != *"$WORDPRESS_CONFIG_EXTRA"* ]]; then
                        # (if the config file already contains the requested PHP code, don't print a warning)
                        echo >&2
                        echo >&2 'WARNING: environment variable "WORDPRESS_CONFIG_EXTRA" is set, but "wp-config.php" already exists'
                        echo >&2 '  The contents of this variable will _not_ be inserted into the existing "wp-config.php" file.'
                        echo >&2 '  (see https://github.com/docker-library/wordpress/issues/333 for more details)'
                        echo >&2
                fi

                # see http://stackoverflow.com/a/2705678/433558
                sed_escape_lhs() {
                        echo "$@" | sed -e 's/[]\/$*.^|[]/\\&/g'
                }
                sed_escape_rhs() {
                        echo "$@" | sed -e 's/[\/&]/\\&/g'
                }
                php_escape() {
                        local escaped="$(php -r 'var_export(('"$2"') $argv[1]);' -- "$1")"
                        if [ "$2" = 'string' ] && [ "${escaped:0:1}" = "'" ]; then
                                escaped="${escaped//$'\n'/"' + \"\\n\" + '"}"
                        fi
                        echo "$escaped"
                }
                set_config() {
                        key="$1"
                        value="$2"
                        var_type="${3:-string}"
                        start="(['\"])$(sed_escape_lhs "$key")\2\s*,"
                        end="\);"
                        if [ "${key:0:1}" = '$' ]; then
                                start="^(\s*)$(sed_escape_lhs "$key")\s*="
                                end=";"
                        fi
                        sed -ri -e "s/($start\s*).*($end)$/\1$(sed_escape_rhs "$(php_escape "$value" "$var_type")")\3/" wp-config.php
                }

                set_config 'DB_HOST' "$WORDPRESS_DB_HOST"
                set_config 'DB_USER' "$WORDPRESS_DB_USER"
                set_config 'DB_PASSWORD' "$WORDPRESS_DB_PASSWORD"
                set_config 'DB_NAME' "$WORDPRESS_DB_NAME"
                set_config 'DB_CHARSET' "$WORDPRESS_DB_CHARSET"
                set_config 'DB_COLLATE' "$WORDPRESS_DB_COLLATE"

                for unique in "${uniqueEnvs[@]}"; do
                        uniqVar="WORDPRESS_$unique"
                        if [ -n "${!uniqVar}" ]; then
                                set_config "$unique" "${!uniqVar}"
                        else
                                # if not specified, let's generate a random value
                                currentVal="$(sed -rn -e "s/define\(\s*(([\'\"])$unique\2\s*,\s*)(['\"])(.*)\3\s*\);/\4/p" wp-config.php)"
                                if [ "$currentVal" = 'put your unique phrase here' ]; then
                                        set_config "$unique" "$(head -c1m /dev/urandom | sha1sum | cut -d' ' -f1)"
                                fi
                        fi
                done

                if [ "$WORDPRESS_TABLE_PREFIX" ]; then
                        set_config '$table_prefix' "$WORDPRESS_TABLE_PREFIX"
                fi

                if [ "$WORDPRESS_DEBUG" ]; then
                        set_config 'WP_DEBUG' 1 boolean
                fi

                if ! TERM=dumb php -- <<'EOPHP'
<?php
// database might not exist, so let's try creating it (just to be safe)

$stderr = fopen('php://stderr', 'w');

// https://codex.wordpress.org/Editing_wp-config.php#MySQL_Alternate_Port
//   "hostname:port"
// https://codex.wordpress.org/Editing_wp-config.php#MySQL_Sockets_or_Pipes
//   "hostname:unix-socket-path"
list($host, $socket) = explode(':', getenv('WORDPRESS_DB_HOST'), 2);
$port = 0;
if (is_numeric($socket)) {
        $port = (int) $socket;
        $socket = null;
}
$user = getenv('WORDPRESS_DB_USER');
$pass = getenv('WORDPRESS_DB_PASSWORD');
$dbName = getenv('WORDPRESS_DB_NAME');

$maxTries = 10;
do {
        $mysql = new mysqli($host, $user, $pass, '', $port, $socket);
        if ($mysql->connect_error) {
                fwrite($stderr, "\n" . 'MySQL Connection Error: (' . $mysql->connect_errno . ') ' . $mysql->connect_error . "\n");
                --$maxTries;
                if ($maxTries <= 0) {
                        exit(1);
                }
                sleep(3);
        }
} while ($mysql->connect_error);

if (!$mysql->query('CREATE DATABASE IF NOT EXISTS `' . $mysql->real_escape_string($dbName) . '`')) {
        fwrite($stderr, "\n" . 'MySQL "CREATE DATABASE" Error: ' . $mysql->error . "\n");
        $mysql->close();
        exit(1);
}

$mysql->close();
EOPHP
                then
                        echo >&2
                        echo >&2 "WARNING: unable to establish a database connection to '$WORDPRESS_DB_HOST'"
                        echo >&2 '  continuing anyways (which might have unexpected results)'
                        echo >&2
                fi
        fi

        # now that we're definitely done writing configuration, let's clear out the relevant envrionment variables (so that stray "phpinfo()" calls don't leak secrets from our code)
        for e in "${envs[@]}"; do
                unset "$e"
        done
fi

exec "$@"

```






    - Build the docker image and push this image to any docker repository

```
docker build -t wordpress .
docker tag wordpress eslamanwar/wordpress:1
docker push eslamanwar/wordpress:1
```



## Deploy to kubernetes
- Create deployment file to deploy the image we just created
- here we set the environmental variables to connect to the same legacy app mysql Database
- kubectl create -f ./wordpress/wordpress-deployment.yaml

wordpress-deployment.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  replicas: 2               ## number of wordpress instances/containers
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      affinity:
        podAntiAffinity:    ## this will force the 2 wordpress containers to run in different worker nodes
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - wordpress
            topologyKey: "kubernetes.io/hostname"
      containers:
      - image: eslamanwar/wordpress:5           ## the image we just created
        name: wordpress
        env:                                    ## set the ENV varables to connect to legacy app
        - name: WORDPRESS_DB_HOST
          value: 192.168.0.40
        - name: WORDPRESS_DB_PASSWORD
          value: Password123
        - name: WORDPRESS_DB_NAME
          value: wordpress
        - name: WORDPRESS_DB_USER
          value: wp_svc
        ports:
        - containerPort: 80
          name: wordpress

```


- Create the k8s service that will loadbalance the traffic accross the 2 wordpress container that we deployed
- kubectl create -f ./wordpress/wordpress-svc.yaml

```
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
      nodePort: 30000           ## it will open port 30000 in all worker nodes to route traffic to our app
  selector:
    app: wordpress
    tier: frontend
  type: NodePort

```





## Deploy the Nginx-ingress 
- This will act as Reverse proxy to route traffic to the wordpress service we just created above
- To deploy thNginx-ingress is we will use Helm

### Deploy helm on k8s cluster
Steps to install Helm from the master node:

```
cd /tmp
curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get > install-helm.sh
chmod u+x install-helm.sh
./install-helm.sh

install tiller as server side:
kubectl -n kube-system create serviceaccount tiller
kubectl create clusterrolebinding tiller --clusterrole cluster-admin --serviceaccount=kube-system:tiller
helm init --service-account tiller
kubectl get pods --namespace kube-system

install nginx-ingress:
helm install stable/nginx-ingress --name my-release  --set defaultBackend.service.type=NodePort

## edit the controller service to map the external ip
kubectl edit service {Nginx-ingress-controller}
spec:
  externalIPs:
  - 192.168.0.72
  - 192.168.0.73
  - 192.168.0.74

```


### Create ingress rule to route traffic to the wordpress service
- kubectl create -f ./ingress.yaml

ingress.yaml
```
  apiVersion: extensions/v1beta1
  kind: Ingress
  metadata:
    annotations:
      kubernetes.io/ingress.class: nginx
    name: wordpress
    namespace: default
  spec:
    rules:
      - host: www.example.com
        http:
          paths:
            - backend:
                serviceName: wordpress
                servicePort: 80
              path: /

```


## Test 
- on the windows machine edit the hosts file and add the record
192.168.0.72    www.example.com

- so we can now go to the browser and open www.example.com






























































