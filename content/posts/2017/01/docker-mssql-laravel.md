Title: Docker를 이용하여 MSSQL + Laravel 개발하기​
Slug: docker-mssql-laravel
Date: 2017-01-03 19:00
Category: Docker
Tags: docker, mssql, laravel
Author: 오회근
Summary: Docker를 이용하여 멀티플랫폼에서 MSSQL + Laravel을 개발해보자.

지인이 새로운 프로젝트를 시작하면서 Stack을 설계해주었다. Database는 Oracle로 정해진 상황에서 빠른 진행을 위해 Python을 사용하기로 하고 개발환경을 구축하였다. 하지만 갑작스럽게 Database가 MSSQL로 변경되면서 Language를 변경하게 되었는데 그나마 Linux에서 오랫동안 MSSQL과 궁합을 맞춰왔던 PHP를 선택하게 되었고 Laravel Framework를 이용하기로 했다.

배포될 Server는 CentOS로 결정되어 있는 상황이었고 개발 환경은 Windows였는데 정작 개발 환경을 만들어 줄 나는 Mac을 사용하고 있다.
이런 상황에서 Docker를 이용하여 서로 다른 OS위에서 동일한 개발 / 배포 환경을 가질 수 있었다.

MSSQL + Laravel + Docker라는 특이한 구조지만 MSSQL를 이용하는게 까다로울 뿐이지 다른 Database로 변경하는 것은 쉽게 가능하므로 Mysql이나 PostgreSQL등을 사용하길 원하는 사람들도 충분히 참고가 될 것 같다.

디렉토리의 구조는 다음과 같다.

![tree](https://cdn-images-1.medium.com/max/1600/1*1XrSHdp2R3xJ0O7Di_wjuA.png)

3개의 Docker 로 구성하였고 이를 Docker Compose를 이용하여 관리한다.

Compose를 이용하면 3개의 도커를 각자의 옵션을 이용해서 일일이 실행하지 않아도 되고 link를 이용하여 각 Container끼리 쉽게 연결할 수 있다.

![structure](https://cdn-images-1.medium.com/max/1600/1*NRaV25nytsp_ZGCnBPxUrg.png)

구성된 모습을 간단하게 도형으로 표현했다.
MSSQL, NGINX, PHP-FPM 3개의 Container를 Compose로 관리하고 Docker Host의 `www` Volume은 NGINX, PHP-FPM Container, `data` Volume은 MSSQL Container에서 공유한다.

다만 MSSQL에서 사용하는 `data` Volume은 Mac에서는 사용할 수가 없다. 아직 MSSQL이 Mac Filesystem을 지원하지 않는다.
완성된 소스는 다음 링크에서 받을 수 있다.

[https://github.com/harryoh/docker-mssql-laravel](https://github.com/harryoh/docker-mssql-laravel)

각 Image대로 Directory를 모두 생성할 필요는 없지만 docker-compose.yml을 열어보지 않고 Directory 구조만으로도 어떤 Docker Image를 사용하는지 알 수가 있어서 선호하는 편이다.

```bash
# project root directory를 만든다.
$ mkdir docker-mssql-laravel && cd $_

# container directory를 만든다.​
$ mkdir nginx && mkdir php && mkdir mssql
```

Project Root Directory를 생성하고 각 Container Directory를 생성했으면 각 Directory에서 Image를 생성하기 위해 Dockerfile을 만든다.​

Docker Container를 실행하기 위해서 docker hub에 있는 Image를 이용할 수 있지만 나는 Dockerfile을 사용하여 새롭게 Build하였다.

## PHP Image 생성​

Docker Image를 생성하기 위해서는 Dockerfile을 작성하고 Build를 실행한다.

**php/Dockerfile**

```dockerfile
FROM ubuntu:16.04
RUN apt-get update && apt-get -y install apt-utils && \
    locale-gen en_US.UTF-8 && update-locale
ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8
ENV DEBIAN_FRONTEND noninteractive
RUN apt-get -y upgrade && apt-get update --fix-missing
RUN apt-get -y install php7.0-fpm php7.0-dev mcrypt php7.0-mcrypt php7.0-mbstring php-pear apt-transport-https composer unzip
RUN sed -i "s/display_errors = Off/display_errors = On/" /etc/php/7.0/fpm/php.ini && \
    sed -i "s/upload_max_filesize = .*/upload_max_filesize = 10M/" /etc/php/7.0/fpm/php.ini && \
    sed -i "s/post_max_size = .*/post_max_size = 12M/" /etc/php/7.0/fpm/php.ini && \
    sed -i "s/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/" /etc/php/7.0/fpm/php.ini && \
    sed -i -e "s/pid =.*/pid = \/var\/run\/php7.0-fpm.pid/" /etc/php/7.0/fpm/php-fpm.conf && \
    sed -i -e "s/error_log =.*/error_log = \/proc\/self\/fd\/2/" /etc/php/7.0/fpm/php-fpm.conf && \
    sed -i -e "s/;daemonize\s*=\s*yes/daemonize = no/g" /etc/php/7.0/fpm/php-fpm.conf && \
    sed -i "s/listen = .*/listen = 9000/" /etc/php/7.0/fpm/pool.d/www.conf && \
    sed -i "s/;catch_workers_output = .*/catch_workers_output = yes/" /etc/php/7.0/fpm/pool.d/www.conf
RUN curl https://packages.microsoft.com/keys/microsoft.asc | apt-key add - && \
    curl https://packages.microsoft.com/config/ubuntu/16.04/prod.list > /etc/apt/sources.list.d/mssql-release.list
RUN apt-get update && \
    ACCEPT_EULA=Y apt-get -y install msodbcsql && \
    apt-get -y install unixodbc-dev-utf16 && \
    pecl install sqlsrv pdo_sqlsrv
RUN echo "extension=/usr/lib/php/20151012/sqlsrv.so" >> /etc/php/7.0/fpm/php.ini && \
    echo "extension=/usr/lib/php/20151012/pdo_sqlsrv.so" >> /etc/php/7.0/fpm/php.ini && \
    echo "extension=/usr/lib/php/20151012/sqlsrv.so" >> /etc/php/7.0/cli/php.ini && \
    echo "extension=/usr/lib/php/20151012/pdo_sqlsrv.so" >> /etc/php/7.0/cli/php.ini
WORKDIR /var/www
EXPOSE 9000
CMD ["php-fpm7.0"]
```

내용은 복잡해 보이지만 실제로 사용된 Keyword는 FROM, RUN, ENV, WORKDIR, EXPOSE, CMD, 총 6개뿐이고 Keywork만으로도 충분히 그 역할을 짐작할 수가 있다.

위의 내용을 간단하게 정리해보면,
ubuntu 16.04 이미지를 받아와서 필요한 패키지들과 mssql driver를 설치하고 php 설정을 변경한다. 9000번 포트로 외부와 연결하고 마지막으로 php-fpm7.0 이라는 프로그램을 실행한다.

docker build 명령을 통해서 빌드할 수도 있지만 나중에 docker-compose를 이용해서 빌드할 예정이다.​

만약 MSSQL을 사용하지 않고 다른 Database를 사용할 예정이라면, 여기서 mssql driver 대신에 해당하는 driver를 설치하면 된다.

## Nginx Image 생성​

nginx는 docker hub의 이미지를 이용하고 docker-compose에서 volume section을 이용하여 동일하게 구현이 가능하다. 하지만 위에서 급한 것처럼 사용하는 Docker를 직관적으로 보일 수 있게 하기 위해서 따로 Directory를 생성하여 Dockerfile로 Build하도록 했다.

Nginx에서 사용할 Configuration 파일을 만든다.

**nginx/default.conf**

```nginx
server {
    listen 80;
    index index.php index.html;
    server_name php-docker.local;
    error_log /var/log/nginx/error.log;
    access_log /var/log/nginx/access.log;
    root /var/www/public;
location / {
        try_files $uri /index.php?$args;
    }
location ~ \.php$ {
        try_files $uri =404;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass php:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }
}
```

위의 내용도 복잡해 보이지만 크게 어려운 내용은 아니다. 핵심은 **php를 호출하는 요청은 php:9000으로 연결하는 것**이다.

여기서 주소 대신 php라는 별칭을 사용하는 이유는 Docker Container는 생성할때에 IP 주소를 자동으로 설정한다. 따라서 어떤 IP를 갖게 될지 미리 알지 못한다. 그렇기 때문에 Nginx Container에서 php Container를 찾지 못해 연결해 줄 수가 없는 것이다.

docker-compose에서는 links section을 이용하여 host name을 넘겨 이 문제를 해결한다. docker-compose는 links에 정의된 Container의 이름과 IP 주소를 hosts 파일에 기록한다.

그렇게 `php:9000` 의 주소는 php container의 주소가 맵핑된다.

**nginx/Dockerfile**

```dockerfile
FROM nginx:latest

COPY default.conf /etc/nginx/conf.d/

WORKDIR /var/www
```

## MSSQL Image

Microsoft에서는 2016년 3월 8일 Linux용 MSSQL Preview 버젼을 발표했다.
시험해본 결과 아직 완벽하지는 않지만 개발용으로는 충분히 사용이 가능하다.​

PHP와 Mssql의 궁합이 썩 좋지는 않지만 특히 SI쪽에서는 어쩔수 없이 이런 조합이 필요할 때가 있다.

[https://hub.docker.com/r/microsoft/mssql-server-linux/](https://hub.docker.com/r/microsoft/mssql-server-linux/) 에 보면 Microsoft에서 공개한 Linux용 MSSQL Docker Image가 있다.

**mssql/Dockerfile**

```dockerfile
FROM microsoft/mssql-server-linux:latest

ENV SA_PASSWORD=Password1234
ENV ACCEPT_EULA=Y

# sql server setup
RUN /opt/mssql/bin/sqlservr-setup --accept-eula --set-sa-password

# set locale
RUN apt-get update && locale-gen en_US.UTF-8 && update-locale

ENV LANG en_US.UTF-8
ENV LANGUAGE en_US:en
ENV LC_ALL en_US.UTF-8
EXPOSE 11433
```

사용 동의를 위해서 ACCEPT_EULA 환경변수를 설정하고 sa password를 설정한다. 원하는 password 넣을 수가 있는데 너무 간단한 Password를 넣으면 MSSQL에서 거부될 수가 있다. 최소 대문자,소문자,숫자 하나이상에 총 8자이상을 넣으면 된다.

원래는 mssql tools을 설치하려 했으나 일시적(?)으로 관련 패키지들이 업그레이드하면서 설치가 되지 않고 있다. 나중에 설치가 되면 docker-compose에서 환경변수를 통해 사용할 Database를 설정하고 mssql container에서 자동으로 database를 생성 및 설정할 수 있는 코드가 들어갈 수가 있다.

## Docker Compose

![docker_compose](https://cdn-images-1.medium.com/max/1600/1*NRaV25nytsp_ZGCnBPxUrg.png)

위에서 사용하였던 간단한 구성도이다. Docker 세개는 이미 구성했는데 각 Container끼리의 관계와 Host와 Volume을 설정해야한다.

**docker-compose.yml**

```dockerfile
version: '2'
services:
    nginx:
        build:
            context: nginx
        ports:
            - "8080:80"
        volumes:
            - ../www/:/var/www
        links:
            - php
    php:
        build:
            context: php
        volumes:
            - ../www/:/var/www
        links:
            - mssql
    mssql:
        build:
            context: mssql
        ports:
            - "11433:1433"
# mac에서는 volumes section을 주석처리해야한다.
        volumes:
            - ./mssql/data:/var/opt/mssql/data
```

위의 구성도 대로 만들어진 docker-compose 파일이다.

nginx와 php 서비스에서는 ../www/ 디렉토리와 volumes을 공유하고 있다.
해당 디렉토리에 Laravel 프로젝트가 생성될 예정이다.

특이사항은 Mac 사용자는 반드시 mssql의 volumes 부분을 주석처리해야한다. 아직까지 MSSQL에서 Data Volume이 Mac의 Filesystem을 지원하지 않아 정상적으로 실행이 되지 않는다.​​

> Volume mapping for Docker-machine on Mac with the SQL Server on Linux image is not supported at this time.

Mac일 경우에는 아직까지는 어쩔 수 없이 Docker Instance를 삭제할 경우 설정된 DB도 사라질 수가 있으니 유의해야한다.

그리고 만약 Mac이나 Windows에서 Docker를 실행하는 것이라면 Docker 설정에서 메모리를 4G 이상으로 올려주어야 실행이 가능한다.

> The default on Docker for Mac and Docker for Windows is 2 GB for the Moby VM, so you will need to change it to 4 GB. The following sections explain how.
> (출처: [https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-setup-docker](https://docs.microsoft.com/en-us/sql/linux/sql-server-linux-setup-docker))

## Docker Compsose 실행

이제는 Container들을 구동시켜본다.

```bash
$ docker-compose up -d
Starting docker_mssql_1
Starting docker_php_1
Starting docker_nginx_1
```

이때 docker-compose.yml 파일이 있는 디렉토리에서 실행해야 한다.

**Database Initial**

정상적으로 실행이 되었으면 Database에 DB를 생성한다.

현재는 mssql-tools 설치가 정상적으로 되고 있지 않아서 DB Client를 이용했는데 DBeaver([http://dbeaver.jkiss.org/](http://dbeaver.jkiss.org/)를 이용하였다.

```
Host: localhost
Port: 11433
User: sa
Pass: Password1234
```

접속에 성공하면 다음과 같은 SQL을 실행한다.

```
CREATE DATABASE testdb;
USE testuser;
CREATE LOGIN testuser WITH PASSWORD = 'TestUser1234';
CREATE USER testuser FOR LOGIN testdb;
ALTER SERVER ROLE sysadmin ADD MEMBER [testuser];
```

이제 MSSQL Server에 testdb 와 testuser 가 생성되었다.

## Laravel Project 생성

Laravel Project를 생성해서 MSSQL와 연결되는 테스트를 해본다.

Laravel Project 생성을 하기 위해서는 그에 맞는 환경이 이미 구성되어야한다. (참고: [https://laravel.com/docs/5.3/installation](https://laravel.com/docs/5.3/installation))

만약 자신의 PC에 구성하기가 어렵다면 Docker Container로 접속하여 간단하게 생성한다.
Laravel Project를 Docker Host에서 생성했다면 Project Configuration으로 건너뛰면 된다. 이미 프로젝트가 생성되어 있는데 또다시 생성할 필요는 없다.

```bash
$ docker exec -it docker_php_1 bash
```

실행중인 docker_php_1 Container로 접속하게 된다.

```bash
root@8d8d1377d394:/var/www# ls -l
total 0
root@8d8d1377d394:/var/www# composer create-project --prefer-dist laravel/laravel .
.
.
.
Writing lock file
Generating autoload files
> Illuminate\Foundation\ComposerScripts::postUpdate
> php artisan optimize
Generating optimized class loader
The compiled class file has been removed.
> php artisan key:generate
Application key [base64:0DizudlmgwFLNUFAOrp/wqFsgCXF9oStKjcBHpeeEx0=] set successfully.
root@8d8d1377d394:/var/www#
```

현재 Directory가 비어 있는 것을 확인하고 laravel project를 생성한다.
직접 Project를 생성하게 되면 자동으로 .env 파일을 생성해주고 application key를 생성해준다.

**Web Browser 확인

Project까지 모두 정상적으로 생성이 되었으면 웹브라우저에서 확인이 가능하다.

`http://localhost:8080`

![laravel](https://cdn-images-1.medium.com/max/1600/1*KYjs0RNskO4FgFXiNGadsA.png)

**Project Configuration**

Project를 직접 생성했다면 .env파일을 자동으로 생성되지만, repository에서 다운로드를 받을 경우는 .env파일을 직접 생성해야한다.

Project Directory로 이동하여 .env 파일을 생성한다.

```bash
$ cd www
$ cp .env.example .env
$ php artisan key:generate
Application key [base64:0DizudlmgwFLNUFAOrp/wqFsgCXF9oStKjcBHpeeEx0=] set successfully.
```

위의 명령은 Docker Host와 Docker Container 아무데서나 상관없다. Docker Host에 laravel 환경이 없다면 Container에서 실행할 수 있다.

이제 application key가 생성되었으면 Database 설정을 한다.

.env 파일의 Database 부분을 다음과 같이 수정한다.

```
DB_CONNECTION=sqlsrv
DB_HOST=mssql
DB_PORT=11433
DB_DATABASE=testdb
DB_USERNAME=testuser
DB_PASSWORD=TestUser1234
```

수정이 완료되었으면 php container를 재시작한다.

```bash
$ docker restart docker_php_1
docker_php_1
```

이제 database migration을 한다.

마찬가지로 docker host든지 container이든지 상관없다.

```
$ php artisan migrate
```

이제 기본적으로 Mssql과 Laravel을 Docker 위에서 작업할 수 있는 환경이 만들어 졌다.

docker-compose.yml이 있는 곳으로 가서 다음과 같이 docker를 중지시킬 수 있다.

```bash
$ docker-compose kill
Killing docker_nginx_1 ... done
Killing docker_php_1 ... done
Killing docker_mssql_1 ... done
```

kill 을 이용하면 Instance들을 삭제하지 않는다. 따라서 다시 docker를 실행하면 Mac 사용하도 Database의 내용을 잃지 않는다.

하지만 down을 사용하면 깨끗하게 instance가 삭제 되므로 Mac 사용자는 조심해야한다.

```
$ docker-compose down
Removing docker_nginx_1 ... done
Removing docker_php_1 ... done
Removing docker_mssql_1 ... done
Removing network docker_default
```

## 마무리

Mac, Linux, Windows 가 docker를 사용하는 방법이 조금씩 틀릴 수가 있지만 대표적인 3가지 OS에서 모두 같은 환경에서 작업할 수 있다는 것은 대단히 유용한 일이다. 뿐만 아니라 함께 개발하는 사람들과 심지어는 배포 조차 같은 환경을 유지할 수 있다.

특히 Cloud와 같은 Scalability, Micro가 중시되는 서비스에서 Docker의 활용범위가 무궁무진하다.

처음에는 Docker가 익숙치 않고 왜 필요한지를 잘 느끼지 못할 수도 있지만 억지로라도 사용하다보면 얼마나 많은 장점들이 있고 편리한지를 자연스럽게 알게 될 것이다.

내 노트북 한대에는 각자의 독립된 서비스를 가진 프로젝트들이 명령어 한줄로 구동되며 심지어는 동시에 구동되어도 아무런 문제가 없다.

그렇게 개발된 코드는 CI를 통해서 배포파일이 생성되고 AWS의 Beanstalk가 몇개인지 모를 EC2 Instance를 Docker를 이용하여 무중단 배포하고 있다.