---
layout: post
title: 라즈베리파이로 워드프레스 Web Server 구축하기
subtitle: 파이 구매부터 도메인 연동까지
tags: [website, rpi]
comments: true
---

갑자기 블로그를 해보고 싶어서 알아보다가 워드프레스라는 좋은 블로그 툴을 알게 되었다. 그러나 워드프레스에서 블로그를 만들면 wordpress.com 가 붙은 도메인만 무료로 쓸 수 있다. 일년에 $96금액을 내야지만 자기 도메인을 연동 할 수있는데 (도메인 가격은 당연히 미포함), 이금액을 아끼면서 라즈베리파이 공부겸 서버를 집에서 구축해 보기로 했다. 이글을 쓰고있는 블로그가 내가 구축한 웹서버에 돌아가고 있는 것이다.

구매부터 라즈베리 세팅까지 했기 때문에 매우 많은 웹사이트를 찾아 다녔지만. 나같이 처음 구매부터 웹서버 완성까지 정리해 준 글이없어서 리소스 정리겸 이글을 쓰게 되었다.

이글에서 다룰 내용은 다음과 같다:

라즈베리파이 구매
라즈베리안 설치방법
웹서버 관리에 유용한 툴 설치 – 방화벽, 트래픽량 감시
웹서버에 워드프레스 구축하기 (LAMP)
도메인 등록 + WordPress 연결방법
SSL 인증 설치 – Let’s Encrtpt

참고 사항은 1,2,4,5 내용은 꼭 순서대로 해야 하지만, 3,6 는 라즈베리안 설치후 아무때나 해도 상관이 없다.


# 1. 라즈베리파이 구매

라즈베리파이는 아마존에서 구매 하였고. 서버로 돌릴것이기 때문에 발열이 있을 수도 있으므로 팬이 달려있는 케이스를 구매 하기로 했다.

비교적 저렴하고 리뷰도 좋길래 별다른 생각없이 이 케이스를 샀다. 내가 직접 레이어별로 조립하는 방식인데 다른 케이스도 이런방식인지는 모르겠지만 라즈베리파이에 꼭 맞게 디자인 되어 충격에 안전하고 팬 소음도 거의 없어 좋다.


# 2. 라즈베리안 설치방법

## OS 고르기

내가 처음에 조금 혼란스러웠던 부분은 라즈베리안이라는 os의 종류가 한개가 아니였다는 점이다.

raspbian desktop 버전은 모니터, 마우스 와 키보드를 라즈베리에 직접 연결해 데스크탑 컴퓨터 처럼쓸 수 있다.

Lite 버전은 headless 라고 불리는 터미널기반 버전이고 ssh를 이용해 자기의 개인컴퓨터로 접속하는 환경이다. 나는 서버 컴퓨터로만 사용할 것 이기때문에 아무래도 전기료를 조금이라도 아낄수 있는 Lite버전을 사용했다.

raspbain 이미지를 SD 카드에 옮기는방법은 공식사이트를 참조하거나 구글에 잘 설명해 놓은 포스팅이 많이 있으니 내용은 스킵한다.

- [공식라즈베리안 다운로드 사이트](https://www.raspberrypi.org/downloads/raspbian/)
- [라즈베리안 설치방법 사이트 (영문)](https://desertbot.io/blog/headless-raspberry-pi-3-bplus-ssh-wifi-setup)


### Troubleshooting

php나 MySQL을 설치 할때 다음과 같은 메세지가 나오고 MacOS ssh을 통해 라즈베리를 접속하고 있다면 이러한 해결방법이 있다.

```
perl: warning: Setting locale failed.
perl: warning: Please check that your locale settings:
      LANGUAGE = (unset),
      LC_ALL = (unset),
      LC_CTYPE = "UTF-8",
      LANG = "en_GB.UTF-8"
    are supported and installed on your system.
perl: warning: Falling back to the standard locale ("C").
locale: Cannot set LC_CTYPE to default locale: No such file or directory
locale: Cannot set LC_ALL to default locale: No such file or directory
```

(라즈베리파이가 아닌) 맥 os 에서 다음과 같은 파일을 편집해서

~~~
sudo nano /etc/ssh/sshd_config
~~~
이러한 문장을 찾아 지우거나 코멘트 처리해주면 된다.


## ssh 허용하기

라즈베리안이 넣어진 SD 카드를 파이에 넣기전에 몇가지 설정사항이 더 있다. 2016년부터 라즈베리파이의 ssh default 가 disable로 설정 되있기때문에 이미지를 SD카드에 옮긴후 몇가지 작업을 더 해줘야한다. ssh 설정을 enable로 바꾸기위해서는 sd카드 메인 루트에 ssh 이름을 가진 빈파일을 만들어 주면된다.

맥이나 리눅스 사용자라면
```console
touch /Volumes/boot/ssh
```
윈도우 사용자라면 txt 에디터를 이용해 (i.e. notepad) ssh 라는 이름을 가진 빈파일을 해당폴더에 넣어주면 된다. ssh.txt 가되지않도록 주의하자.


## 네트워크 정보 입력하기

라즈베리파이를 처음으로 켰을때 자동적으로 인터넷에 접속하게 하기위해서는 네트워크 정보를 미리 입력해 줘야한다.

```console
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

network={
    ssid="NETWORK-NAME"
    psk="NETWORK-PASSWORD"
}
```

자기의 나라 코드에 맞게 country=US 의 US부분을 편집해주고. network={} 안에있는 부분에 자기의 network 이름과 비밀번호를 바꿔주자.

그다음 아래의 이름을 가진 파일을 만들어서 그 안에 넣은후 저장을 해준다. ssh파일과 똑같이 메인 boot폴더에 넣어준다.

~~~
wpa_supplicant.conf
~~~

그다음 스텝은 일반적인 라즈베리언 설치방법과 똑같다. SD카드를 eject 해준후 파이에 넣고 부팅을 시켜주면된다. 공식사이트나 다른글에서 많이 설명하고 있기때문에 이글에서 ssh를 이용해 어떻게 파이에 접속하는지는 다루지않는다. 보안을위해 라즈베이파이에 접속후 호스팅네임과 비밀번호를 바꾸는 것을 잊지 말자!


# 3. 웹서버 관리에 유용한 툴 설치 – 방화벽, 트래픽량 감시

## 방화벽 설치

서버의 보안을 위해 방화벽(firewall)을 설치하고 만약을 위해 트래픽량을 감시하는 툴을 설치 하기로 했다.

파이에는 iptables 이라는 방화벽 솔루션이 이미 있지만 disable이 디폴트로 설정되있다. ufw라는 툴을 이용해 iptables를 복잡하지않게 설정해 주자.

[이 링크](https://www.raspberrypi.org/documentation/configuration/security.md)로 들어가 install a firewall 부분을 따라해 주면된다.

내가 설정한 부분은 21(ftp), 22(ssh), http(80), https(443) 이다.

```console
sudo ufw allow 21/tcp
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
Troubleshooting
```

### Troubleshooting

설치후 다음과 같은 오류가 나왔을때 해결 방법을 소개 하겠다.

~~~
[Errno 2] iptables v1.6.0: can't initialize iptables table `filter': Table does not exist (do you need to insmod?)
Perhaps iptables or your kernel needs to be upgraded.
~~~

라즈베리파이 커널을 업데이트 하는 명령
```console
sudo apt-get install --reinstall raspberrypi-bootloader raspberrypi-kernel
```
그리고 간단히 reboot해준다

```console
sudo reboot
```

## 트래픽량 확인하기

트래픽 감시툴을 설치 + 이용하는 방법은 [이블로그](https://kr.minibrary.com/10/) 글을 참고했다


# 4. 웹서버에 워드프레스 구축하기 (LAMP)

웹서버를 구축하는 방법은 다양하지만 나는 Apache와 MySQL를 이용하기로 했다. 별다른 이유는 없고 라즈베리파이 [공식 웹사이트](https://projects.raspberrypi.org/en/projects/lamp-web-server-with-wordpress/8)에 있는 메뉴얼이 이두개를 이용해서 설치했기때문이다. 나는 Lite버전을 이용해서 몇몇 테스트 부분을 스킵한것을 빼놓고는 그대로 따라했다.

# 5. 도메인 등록 + WordPress 연결방법

이제 라즈베리파이의 서버 ip 주소를 도메인을 등록해 forwarding 하는 작업이다. 나는 도메인을 [ionos.com](http://ionos.com/) 라는 사이트에서 구매하게되었는데. 별다른 이유는 없고 $1 에 처음 일년을 쓸수있다는 말에 혹해서 이용하게 되었다.

호스팅하는 방법은 [이 블로그](https://www.e-tinkers.com/2016/11/hosting-wordpress-on-raspberry-pi-part-5-dedicated-ip-domain-name-and-dns/)를 주로 참고 하였는데 다른것은 다 따라할수 있었지만 한가지 다른 점은 Dynamic DNS을 할때였다. 내가 쓰는 ionos에서는 DDNS를 제공하지않기 때문에 따로 [Dynu.com](https://www.dynu.com/) 라는 무료 DDNS 사이트를 이용했다.

Dynu.com에서 어떻게 DDNS를 등록 하는지에 대해서는 [여기서](https://www.dynu.com/DynamicDNS/IPUpdateClient/RaspberryPi-Dynamic-DNS) 알아 볼수 있다.


# 6. SSL 인증 설치 – Let’s Encrypt

HTTP (port 80)대신 HTTPS(443) 통신을 이용하면 통신되는 데이터가 암호화가되어 전송이 된다. 혹시모를 데이터 탈취를 막고 SSL인증서를 통해 나의 웹사이트가 안전하다는것을 알리기위해 무료 SSL 발급 툴을 설치해 주었다. 설치방법은 [이 사이트](https://pimylifeup.com/raspberry-pi-ssl-lets-encrypt/)를 참고했다. 설치후 호스팅 스텝중에 하나였던 Router Port forwarding 에 443 port 추가를 잊지말자.


# 마무리

이로써 라즈베리파이로 워드프레스 웹서버 구축 하기가 끝났다. 간단한 Linux 사용법만 알면 비교적 짧은 시간에 자기만의 서버를 완성할 수 있는 재밌는 프로젝트였는데. 이제 나만의 웹서버가 만들어졌으니 본격적으로 home IoT를 구현해볼 차례인가보다.


# 그외 유용한 설치 목록

git – 내 pc 간의 파일 이동이나, 인터넷에있는 프로젝트를 손쉽게 다운받을 수 있는 Version Control 시스템
```console
sudo apt install git-all
```
그외 추가예정…


# Reference

- https://desertbot.io/blog/headless-raspberry-pi-3-bplus-ssh-wifi-setup
- https://www.raspberrypi.org/documentation/configuration/security.md
- https://www.e-tinkers.com/2016/11/hosting-wordpress-on-raspberry-pi-part-5-dedicated-ip-domain-name-and-dns/
- https://www.dynu.com/DynamicDNS/IPUpdateClient/RaspberryPi-Dynamic-DNS
- https://pimylifeup.com/raspberry-pi-ssl-lets-encrypt/
