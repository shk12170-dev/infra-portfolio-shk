# Linux 서버 기본 실습 기록 (WSL2 Ubuntu)

- 실습일: 2026-11-22
- 환경: Windows 11 위의 **WSL2 가상 환경**, Ubuntu 26.04.1 LTS, systemd 사용. 서버 장비나 VMware 가상 머신이 아니라 개인 PC의 WSL2 환경입니다.
- 목적: 서버 운영의 기본인 **계정·권한 설정, 패키지 설치, 서비스 시작·중지**를 직접 실행하고 결과를 확인합니다.
- 범위: Windows Server와 스토리지는 아직 해 보지 않았습니다. 이 실습은 Linux 기본 작업만 다룹니다.

## 한눈에 보기

| 작업 | 한 일 | 확인한 결과 |
| --- | --- | --- |
| 계정·그룹 | 그룹 `infra-team` 생성, 사용자 `labuser`(그룹 소속)와 `otheruser`(그룹 밖) 생성, `labuser` 비밀번호 잠금 | `id`로 그룹 소속 확인, 잠금 상태(`L`) 확인 |
| 디렉터리 권한 | `/srv/lab-share`를 `root:infra-team`, `2770`(그룹 쓰기, 그룹 상속)으로 설정 | `labuser`는 파일 생성 성공, `otheruser`는 `Permission denied` |
| 패키지 설치 | `apt`로 nginx 설치 | `nginx -v`, `dpkg -l`로 설치 확인(1.28.3) |
| 서비스 시작·중지 | `systemctl stop`/`start`로 nginx 중지·시작 | 활성 상태에서 HTTP 200, 중지 상태에서 연결 실패(HTTP 000), 재시작 뒤 HTTP 200 |
| 로그 확인 | `journalctl -u nginx`로 중지·시작 기록 확인 | 중지와 시작 이벤트가 시간순으로 남음 |
| 정리 | nginx 삭제, 계정·그룹·디렉터리 삭제 | 실습 전 상태로 복구 |

## 실행 기록 전체

아래는 실제 실행 결과 그대로입니다(호스트 이름만 `lab-host`로 바꿨습니다).

```text
# 환경

$ grep PRETTY_NAME /etc/os-release; uname -r; ps -p 1 -o comm=
PRETTY_NAME="Ubuntu 26.04.1 LTS"
6.18.33.2-microsoft-standard-WSL2
systemd

## 1. 계정·그룹

$ groupadd infra-team && useradd -m -s /bin/bash -G infra-team labuser && useradd -m -s /bin/bash otheruser

$ id labuser; id otheruser
uid=1001(labuser) gid=1003(labuser) groups=1003(labuser),1002(infra-team)
uid=1002(otheruser) gid=1004(otheruser) groups=1004(otheruser)

$ passwd -l labuser; passwd -S labuser
passwd: password changed.
labuser L 2026-11-22 0 99999 7 -1

## 2. 디렉터리 권한(그룹 공유)

$ mkdir /srv/lab-share && chown root:infra-team /srv/lab-share && chmod 2770 /srv/lab-share && ls -ld /srv/lab-share
drwxrws--- 2 root infra-team 4096 Nov 22 17:14 /srv/lab-share

$ sudo -u labuser bash -c "echo ok > /srv/lab-share/a.txt" && ls -l /srv/lab-share
total 4
-rw-r--r-- 1 labuser infra-team 3 Nov 22 17:14 a.txt

$ sudo -u otheruser bash -c "cat /srv/lab-share/a.txt"
cat: /srv/lab-share/a.txt: Permission denied

## 3. 패키지 설치

$ apt-get update -qq && apt-get install -y -qq nginx >/dev/null; nginx -v; dpkg -l nginx | tail -1
nginx version: nginx/1.28.3 (Ubuntu)
ii  nginx          1.28.3-2ubuntu1.14 amd64        small, powerful, scalable web/proxy server

## 4. 서비스 시작·중지

$ systemctl is-active nginx; curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1/
active
HTTP 200

$ systemctl stop nginx; systemctl is-active nginx; curl -s -o /dev/null -w "HTTP %{http_code}\n" --max-time 3 http://127.0.0.1/ || echo "연결 거부(서비스 중지 상태)"
inactive
HTTP 000
연결 거부(서비스 중지 상태)

$ systemctl start nginx; systemctl is-active nginx; curl -s -o /dev/null -w "HTTP %{http_code}\n" http://127.0.0.1/
active
HTTP 200

$ systemctl is-enabled nginx; journalctl -u nginx --no-pager -n 4 | cut -c1-110
enabled
Nov 22 17:14:36 lab-host systemd[1]: nginx.service: Deactivated successfully.
Nov 22 17:14:36 lab-host systemd[1]: Stopped nginx.service - A high performance web server and a revers
Nov 22 17:14:36 lab-host systemd[1]: Starting nginx.service - A high performance web server and a rever
Nov 22 17:14:36 lab-host systemd[1]: Started nginx.service - A high performance web server and a revers

## 5. 정리(원상 복구)

$ systemctl stop nginx; apt-get purge -y -qq nginx nginx-common >/dev/null 2>&1; apt-get autoremove -y -qq >/dev/null 2>&1; userdel -r labuser; userdel -r otheruser; groupdel infra-team; rm -rf /srv/lab-share; dpkg -l nginx 2>&1 | tail -1; id labuser 2>&1; ls /srv
userdel: labuser mail spool (/var/mail/labuser) not found
userdel: otheruser mail spool (/var/mail/otheruser) not found
dpkg-query: no packages found matching nginx
id: 'labuser': no such user
```

## 한계와 다음 계획

- 한 대의 가상 환경에서 기본 작업을 한 것이며, 여러 대 서버 구성이나 네트워크 장비 연동은 아닙니다.
- 다음 계획: Windows Server 평가판을 가상 머신에 설치해 기본 구성(역할·계정·디스크)을 익히고, 스토리지는 RAID·볼륨 기초 개념부터 정리합니다.
