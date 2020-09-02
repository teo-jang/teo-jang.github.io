---
title:  "Ubuntu server에 Visual studio code server 설치하기"
categories:
    - Infra
tags:
    - Code, Server
---

#### 들어가며

기존에 Jupyter lab 원격 접속, Google Colaboratory 등을 이용하여 원격으로 작업을 하다가 최근 golang에 약간 흥미가 생겨 IDE를 iPad에서 이용할 수 있을까하는 생각을 하게 되었다.

그러다 업무차 사내 시스템에 Code server가 설치되어 있는 것이 생각났고, 자체적으로 구축하여 이용해보고자 한다.

설치 방법은 크게 2가지로 나눌 수 있다.
- Docker를 이용해 Code Server container를 생성하여, container 안에서 작업을 한다.
- Ubuntu server에 직접 설치를 한다.

---

#### Docker를 통한 Code-server container 설치
아래 URI를 참조하여 설치하면 된다.

https://hub.docker.com/r/linuxserver/code-server

Docker가 설치되어 있지 않다면 설치를 한다.
(Ubuntu 18.04 LTS 기준)

```bash
#APT Update
$ sudo apt-get update
$ sudo apt-get upgrade
#Install docker
$ sudo apt install docker.io
#systemd에 docker 등록
$ sudo systemctl start docker
$ sudo systemctl enable docker
#docker 상태 확인
$ sudo systemctl status docker
```

Docker가 무사히 설치가 되었음을 확인하면 아래 Command를 실행한다.
executable한 file로 만들어서 실행해도 무관하고, 이대로 shell에 입력해도 된다.

```bash
$ docker create \
  --name=code-server \
  -e PUID=1000 \
  -e PGID=1000 \
  -e TZ=Europe/London \
  -e PASSWORD={your_password} `#optional` \
  -e SUDO_PASSWORD={your_sudo_password} `#optional` \
  -e PROXY_DOMAIN={your_proxy_domain} `#optional` \
  -p 8443:8443 \
  -v /path/to/appdata/config:/config \
  --restart unless-stopped \
  linuxserver/code-server
```

Docker 머신이 무사히 생성되었는지 확인해본다.

```bash
$ sudo docker ps -a
```

status 가 RUNNING인 것을 확인하고, localhost에서 접근이 되는지 우선 확인한다.

```bash
$ wget localhost:8443
```

wget한 파일 내용에 Password를 입력하라는 내용이 얼추 html로 작성되어있으면 정상 작동되니, 방화벽 설정등을 확인하여 외부에서 접속을 시도해본다.

접속이 되지 않는다면 Firewall의 Inbound rule을 확인하거나, OS 자체의 방화벽을 확인한다. ubuntu를 이용한다면 ufw를 통해 설정한 port를 allow한다.

```bash
$ sudo ufw allow 8443
```

---

#### Ubuntu server 직접 설치

Ubuntu server에 직접 설치하는 것은 이래저래 손이 많이 가지만 단 하나의 장점을 꼽는다면 서버가 실수로 맛이 가서 container가 꺼졌을 때 재설정하는 귀찮음을 감수하지 않아도 된다는 것이다.

요즘 N/W나 Computing power 성능이 워낙 좋다보니 이런 것 쯤은 무시할 수 있지만, 맘잡고 접속했을 때 접속하면 마음상하니까 직접 설치했다.

아래 site를 참조하여 글을 작성했다.

https://www.howtoforge.com/how-to-install-code-server-ide-on-ubuntu-1804/

```bash
#code를 위한 계정 생성
$ useradd -m -s /bin/bash code
$ passwd code

#code 계정에 code 설치
$ su - code

#https://github.com/cdr/code-server/releases 에 들어가서 원하는 release 버전을 선택하면 된다. 
#일단 site에 있는 그대로 이용
$ wget https://github.com/cdr/code-server/releases/download/2.1692-vsc1.39.2/code-server2.1692-vsc1.39.2-linux-x86_64.tar.gz

# 압축해제
$ tar -xzf code-server2.1692-vsc1.39.2-linux-x86_64.tar.gz
# 압축 푼 내용을 bin folder로 이름을 변경
$ mv code-server2.1692-vsc1.39.2-linux-x86_64/ bin/
#실행권한 추가
$ chmod +x ~/bin/code-server
#code-server 의 workspace로 쓰일 폴더 추가
$ mkdir -p ~/data
```

우선 다운로드 받아서 설치는 했는데, 잘 되나 테스트나 해보자.

```bash
$ cd ~/bin/
$ ./code-server
```
위와 같이 입력하면 뭐라뭐라 뜨면서, 8080 port로 접근하면 code-server에 접근할 수 있다. shell에 나온 password를 이용하여 접근해보고 원활히 작동하는지 확인한다. 원활히 작동이 된다면 systemd에 등록을 해두는게 여러모로 편리하다.

```bash
$ cd /etc/systemd/system/
$ sudo vim code-server.service
```
code-server.service 파일의 내용은 사이트를 참고하여 아래와 같이 작성한다.

```bash
[Unit]
Description=code-server
After=nginx.service

[Service]
User=code
WorkingDirectory=/home/code
Environment=PASSWORD={YOUR_PASSWORD}
#ExecStart=/home/code/bin/code-server --host 127.0.0.1 --user-data-dir /home/code/data --auth password
ExecStart=/home/code/bin/code-server --user-data-dir /
Restart=always

[Install]
WantedBy=multi-user.target
```
위에서 host 부분을 삭제한 이유는 원격 접속을 위해 구축하는건데, 저렇게 할 경우 host에서의 접근만 allow하기 때문에 삭제했다.

그 후 추가한 service를 systemd에 등록하기 위해 daemon을 reload 한 후, code-server를 실행해보자.

```bash
$ sudo systemctl daemon-reload
$ sudo systemctl start code-server
$ sudo systemctl enable code-server
$ sudo systemctl status code-server
```

Running 상태로 작동하는 것을 확인 후, 외부에서 접근하여 동작하는 것을 확인할 수 있다.

---

#### 마치며

최근 Mosh + Vim 조합이나 Jupyterlab을 이용하다가 처음하는 언어를 어두컴컴한 화면에서 하기 싫어서 설치해봤는데, 과연 어떨지 기대가 된다.