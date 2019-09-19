---
layout: post
title: 라즈베리파이 웹서버에 새로운 도메인을 추가하는 방법
tags: [rpi, website]
comments: true
---

몇일전에 [라즈베리파이로 웹서버를 구축](https://cloebot.github.io/2019-04-06-how-to-set-up-web-server-with-rpi/)한후 이 블로그를 호스팅 할 수 있게 되었다. 동생의 요청으로어떤 웹페이지를 일시적인 기간동안 호스팅 해야할 일 이 생겼는데, 이번 포스팅에서는 기존의 아파치서버에서 어떻게 하나의 서버를 공유하고 새로운 도메인을 추가 하는지 알아보도록 하겠다.

일단 하나의 서버로 여러개의 도메인을 호스팅 하기 위해서는 **Virtual hosting** 이라는 기술을 사용하고 **Virtual Private Server(VPS)** 를 이용해야한다.

VPS의 개념은 간단하다. 80 나 433 포트를 통해 웹요청이 들어 왔을때,  여러개의 같은 공유기를 쓰는 디바이스중 라즈베이파이의 unique 한 ip를 불러오기위해 **Forwarding** 이라는 기술을 사용하는데. VPS도 마찬가지로, 요청은 같은 파이 IP로 들어오지만 **Virtual hosting** 을 통해 도메인에 따라 어떤 directory로 갈지 결정하는 방법을 말한다.

이제 본격적으로 도메인을 추가하는 방법을 시작해보자.

# 1. 웹 데이터 추가하기

`/var/www/` 디렉토리에는 내가 운영하고 있는 사이트의 데이터들이 저장되어 있는 공간이다.  새로운 사이트를호스팅 할 것이므로 여기에 자기가 원하는 폴더를 만들어 소스를 넣어주면 된다:

```
cd /var/www
sudo mkdir gojackson
```
이제 디렉토리안에는 wordpress 소스파일이 있는 `html` 폴더랑 내가 이제 추가할 새로운 폴더가 있을것이다.

`gojackson html`

난 git 을 새로 셋업해서 메인 pc 에있는 소스 파일들을 라즈베리파이로 옮겼는데, 이 방법은 구글에 이미 정보가 많으므로 이번 포스팅에는 생략한다.

# 2. 파일 Permission 정하기

새로운 웹파일을 누가 편집할 수 있나를 정할 차례이다. 많은 옵션이 있지만 나는 지금 파이에 접속하고있는유저라면 파일을 바꿀 수 있도록 설정을 하기로 했다

```
sudo chown -R $USER:$USER /var/www/gojackson
```

그다음엔 웹페이지 접속자가 읽을 수 있는 퍼밋 (편집은 못하도록) 설정을 해야한다:

```
sudo chmod -R 755 /var/www/gojackson`
```

# 3. 새로운 가상 호스트 파일 만들기

이제 아파치 서버가 요청을 받았을때 어떤 도메인으로 가야할지 알려주는 설정을 해보자.

`cd /etc/apache2/sites-available/`
아파치를 설치를 하고 아무 손도 안댔더라면 `000-default.conf` 라는 virtual host 파일이 있을것이다. SSL certificate 을위해 443을 열었다면 나처럼 다른 default 파일들이 있을 것이다

```
000-default-le-ssl.conf
000-default.conf
default-ssl.conf
```

새로운 웹은 일단 ssl 없이 추가 할 것이기 때문에 하나의 conf 파일 만 추가하면 된다. 그리고 기존의디폴트 파일과 새로운 웹의 혼동을 막기위해 기존파일을 복사해서 이름을 바꾸어 주었다:

```
sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/cloelee.org.conf

sudo cp /etc/apache2/sites-available/000-default.conf /etc/apache2/sites-available/gojackson.tk.conf
```

이제 편집기를 이용해서 gojackson.tk.conf의 내용을 다르게 바꿀 차례이다. conf의 내용을 보면 대충이런 컨텐츠가 있을것인데

```
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    DocumentRoot /var/www/html
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

그중에서 내가 편집한 부분은 서버각자의 도메인 이름이랑 디텍토리 이름이다:

```
<VirtualHost *:80>
        <Directory /var/www/gojackson>
                Options Indexes FollowSymLinks MultiViews
                AllowOverride All
                Require all granted
        </Directory>

        ServerName gojackson.tk
        ServerAlias www.gojackson.tk
        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/gojackson

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

RewriteEngine on
RewriteCond %{SERVER_NAME} =gojackson.tk
RewriteRule ^ https://%{SERVER_NAME}%{REQUEST_URI} [END,NE,R=permanent]
```

# 4. 아파치 업데이트 하고 재시작 하기

모든 세팅을 맞추었으니 아파치 설정을 켜주고 재시작해보자:

사이트를 Enable 설정은 `a2ensite` 를 disable 은 `a2dissite` 가 커멘드 명령어이다. 나는 아까 디폴트설정을 복사해서 이름만 바꾸었으니 기존에 있던 설정을 중지하고 새로운 설정들을 enable 해야한다

```
sudo a2dissite 000-default.conf
sudo a2ensite cloelee.org.conf
sudo a2ensite gojackson.tk.conf
```

새로운 설정 업데이트를 위해 재시작하자

`sudo service apache2 restart`


# 5. 마무리

마지막으로 [지난번 포스팅](https://cloebot.github.io/2019-04-06-how-to-set-up-web-server-with-rpi/)에서 다룬 것과 마찬가지로 public ip를 도메인 사이트에 등록하고 DDNS 까지 연결해주면 끝이난다.

이로써 기존에 있던 하나의 웹서버에 도메인을 추가하는 방법을 알아 보았다. 이런방식을 이용하면 하드웨어의 한계에 따라 원하는 도메인 수 만큼 추가 할수 있다. 내가 추가한 웹사이트는 유저인풋이 없는 기본적인 웹사이트여서 ssl를 추가하지는 않지만 certificate 도 각 도메인마다 어렵지 않게 추가 할 수 있을 것이다.

# 참고 사이트
https://www.digitalocean.com/community/tutorials/how-to-set-up-apache-virtual-hosts-on-ubuntu-14-04-lts
