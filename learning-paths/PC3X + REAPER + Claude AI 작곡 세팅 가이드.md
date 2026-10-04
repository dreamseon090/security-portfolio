# PC3X + REAPER + Claude AI 작곡 세팅 가이드 (처음부터)

Oct 4, 2026 · @dream seon

## 전체 그림

이 세팅은 프로그램 네 개가 줄줄이 이어진 구조다. 내가 Claude Desktop에 말로 곡을 설명하면, Claude가 음표를 만들어 REAPER 트랙에 넣고, REAPER가 그 음표를 USB로 PC3X에 보내서 PC3X가 소리를 낸다.

&#91;embedded content: 연결 구조 · 4단계 + 편집기\]

| 프로그램 | 역할 | 비유 |
| --- | --- | --- |
| Claude Desktop | 말을 듣고 드럼·코드·베이스 음표를 만든다 | 작곡가 |
| reaper-mcp | Claude의 명령을 REAPER가 알아듣게 바꿔 준다 | 통역사 |
| reapy 서버 | REAPER 안에서 바깥 명령을 받아 주는 문지기. REAPER를 켤 때마다 켜져 있어야 한다 | 문지기 |
| REAPER | 트랙별 MIDI 악보를 보관하고, 재생하면 PC3X로 보낸다 | 악보 보관함·지휘자 |
| PC3X | 채널별로 다른 악기 소리를 낸다 | 악기 |
| SoundTower 에디터 | PC3X 소리·셋업을 컴퓨터 화면으로 편집한다. 녹음 기능은 없다 | 악기 정비소 |

기억할 것 세 가지:

- USB로는 MIDI(연주 신호)만 오간다. 소리는 PC3X 스피커·헤드폰에서 나고, REAPER 안에는 없다. 그래서 REAPER 믹서 페이더로는 소리 크기가 안 바뀌고, PC3X 슬라이더나 벨로시티로 조절한다.
- SoundTower 에디터와 REAPER는 PC3X USB 포트를 동시에 못 쓴다. 하나를 쓸 땐 다른 하나를 끈다.
- Claude(MCP)는 음표를 하나씩만 넣을 수 있어서 느리다. 곡 단위로는 Claude에게 .mid 파일로 달라고 해서 REAPER에 끌어다 놓는 게 빠르다.

## 한 번만 하는 설치

아래는 실제로 성공한 순서만 정리한 것이다. 중간에 막혔던 방법(enable\_reapy.py, setup\_reapy.py, configure\_reaper)은 이 PC에서 안 됐으니 쓰지 않는다. 명령어는 시작 메뉴에서 cmd를 검색해 연 명령 프롬프트에 한 줄씩 붙여 넣는다.

### 1. Python

이 PC에는 Python 3.14가 `C:\Users\SEON\AppData\Local\Programs\Python\Python314`에 설치돼 있다. 새 PC라면 python.org에서 64-bit를 받고 설치 첫 화면의 **Add python.exe to PATH**에 꼭 체크한다.

확인:

```
python --version
where python
```

`where python` 첫 줄이 Python 폴더 위치다. `WindowsApps\python.exe` 줄은 스토어 바로가기라 무시한다.

### 2. reaper-mcp 설치

```
python -m pip install reaper-mcp-server
where reaper-mcp-server
```

두 번째 명령 결과 `C:\Users\SEON\AppData\Local\Programs\Python\Python314\Scripts\reaper-mcp-server.exe`가 8단계에서 쓸 경로다.

### 3. REAPER 설치

reaper.fm에서 Windows 64-bit를 받아 설치한다. 켤 때 뜨는 "REAPER IS NOT FREE" 창은 평가판 안내다. 60일 동안 기능 제한이 없고, X로 닫거나 카운트가 끝나면 계속 쓰기를 누르면 된다.

### 4. REAPER에서 Python 켜기

1. REAPER에서 **Ctrl + P** → 왼쪽 **Plug-ins → ReaScript**.
2. **Enable Python for use with ReaScript**에 체크한다. 이 체크가 풀려 있어서 한참 막혔었다.
3. **Custom path to Python dll directory**: 폴더까지만 넣는다. 끝에 파일 이름을 붙이지 않는다. `C:\Users\SEON\AppData\Local\Programs\Python\Python314`
4. **Force ReaScript to use specific Python .dll**: `python314.dll` (오타 주의)
5. Apply → OK → REAPER 재시작.
6. 다시 들어가서 Python 줄에 `python314.dll is installed`가 보이고 체크가 유지돼 있으면 성공이다. Actions → New action → Load ReaScript 창의 파일 형식 목록에 `*.py`가 보여야 한다.

### 5. 웹 인터페이스(포트 2307) 추가

reapy가 REAPER를 깨울 때 쓰는 통로다.

1. **Ctrl + P** → 왼쪽 맨 아래 **Control/OSC/web** → **Add**.
2. Control surface mode: **Web browser interface**, Port: **2307**, 나머지는 기본값.
3. OK. 브라우저에서 `http://localhost:2307`을 열었을 때 REAPER 리모컨 화면이 뜨면 된다. Windows 방화벽 창이 뜨면 허용한다.

### 6. reapy 서버 스크립트 등록

AppData는 숨김 폴더라 REAPER가 못 읽을 때가 있어서, 스크립트를 밖으로 복사해 쓴다.

```
mkdir C:\reapy
copy "C:\Users\SEON\AppData\Local\Programs\Python\Python314\Lib\site-packages\reapy\reascripts\activate_reapy_server.py" C:\reapy\
```

1. REAPER **Actions → Show action list** → **New action… → Load ReaScript…**
2. 파일 이름 칸에 `C:\reapy\activate_reapy_server.py`를 붙여 넣고 열기.
3. Filter에 `activate` → **Script: activate\_reapy\_server.py** 선택 → **Run**. 아무 창도 안 뜨면 정상이다.
4. 다시 Run을 누르면 "running in background" 창이 뜬다. 이미 돌고 있다는 뜻이니 **Cancel**을 누른다. Terminate를 누르면 서버가 꺼진다.

확인:

```
python -c "import reapy; print('연결 성공:', reapy.Project().n_tracks, '트랙')"
```

`연결 성공: 0 트랙`처럼 나오면 된다.

### 7. REAPER 켤 때 서버 자동 실행 (선택)

안 하면 REAPER를 켤 때마다 6단계 3번처럼 Run을 눌러야 한다.

1. Actions 창에서 Script: activate\_reapy\_server.py를 오른쪽 클릭 → **Copy selected action command ID** (`_RS`로 시작하는 문자열).
2. REAPER **Options → Show REAPER resource path in explorer** → **Scripts** 폴더.
3. 메모장으로 아래 한 줄을 쓰고 `_RS여기붙여넣기`를 1번 ID로 바꾼다.

```lua
reaper.Main_OnCommand(reaper.NamedCommandLookup("_RS여기붙여넣기"), 0)
```

4. 파일 이름 `__startup.lua`(밑줄 두 개), 파일 형식 모든 파일로 Scripts 폴더에 저장 → REAPER 재시작 → 6단계 확인 명령으로 테스트.

### 8. Claude Desktop에 연결

1. claude.ai/download에서 Claude Desktop 설치 → 로그인.
2. **Settings → Developer → Edit Config** → `claude_desktop_config.json`을 메모장으로 연다. 먼저 전체를 복사해 백업해 둔다.
3. 기존 내용은 지우지 않는다. 맨 아래 `"coworkUserFilesPath": ...` 줄 끝에 쉼표를 붙이고, 마지막 `}` 바로 위에 아래 덩어리를 넣는다.

```json
  "coworkUserFilesPath": "C:\\Users\\SEON\\Claude",
  "mcpServers": {
    "reaper": {
      "command": "C:\\Users\\SEON\\AppData\\Local\\Programs\\Python\\Python314\\Scripts\\reaper-mcp-server.exe",
      "args": []
    }
  }
}
```

경로의 `\`는 JSON에서 `\\` 두 개로 쓴다. 맨 마지막 `}`는 하나만 있어야 한다.

4. 저장 → 작업표시줄 트레이(^)의 Claude 아이콘 오른쪽 클릭 → **Quit** → 다시 실행. 창 X만으로는 안 꺼진다.
5. REAPER와 reapy 서버를 켠 상태에서 Claude Desktop에 "REAPER 프로젝트 정보 알려줘"를 보낸다. 허용 창이 뜨면 **Always allow**. 템포·트랙 수가 나오면 연결 성공이다.

### 9. REAPER에서 PC3X 쓰기 허용

1. PC3X를 USB로 연결하고 켠다. SoundTower 에디터는 끈다.
2. **Ctrl + P → Audio → MIDI Inputs** → PC3 오른쪽 클릭 → **Enable input**.
3. **MIDI Outputs** → PC3 오른쪽 클릭 → **Enable output** → OK.

## PC3X 준비: REAPER Mixer 셋업

REAPER는 트랙마다 PC3X의 다른 채널로 음표를 보낸다. PC3X 쪽에서는 채널마다 악기를 걸어 둬야 한다. 이걸 셋업 하나로 만들어 두면 두 가지가 한 번에 해결된다.

- 셋업을 부르면 채널별 악기가 자동으로 걸린다.
- PC3X 슬라이더 A\~D가 드럼·베이스·로즈·패드 볼륨 페이더가 된다.

| 존 | 프로그램 | Channel | 건반 (Lo\~Hi) | Entry Vol | Status | 볼륨 슬라이더 |
| --- | --- | --- | --- | --- | --- | --- |
| 1 | 113 NYC Kits | 10 | C-1 \~ C-1 | 110 | Active | A |
| 2 | 110 Jaco Fretless | 2 | C-1 \~ C-1 | 110 | Active | B |
| 3 | 18 Stevie's Rhds | 1 | C-1 \~ C-1 | 110 | Active | C |
| 4 | 79 Airy Pad | 3 | C-1 \~ C-1 | 90 | Active | D |

건반을 C-1로 두는 이유: 88건반에 없는 음이라 건반을 쳐도 이 존들은 소리가 안 난다. 페이더 역할만 한다. Status가 Muted면 슬라이더 신호도 안 나가니 반드시 Active로 둔다.

### SoundTower 에디터에서 만들기

1. REAPER를 끄고 SoundTower 에디터를 켠다. PC3X는 Setup 모드로 두고 Edit 버튼은 누르지 않는다.
2. 툴바 **SETUP** → **128 Default Setup** → **EDIT**.
3. **Zone 1** 선택 → 프로그램 칸에서 113, **Channel** 10, 건반 그림 양 끝 칸(A0, C8)을 둘 다 **C-1**, **Entry Vol** 110, **Status** Active. 오른쪽 **Vel Zone**은 127 / 1 그대로 둔다.
4. 툴바 **Edit → Add New Zone**으로 존 2\~4를 추가하고 표대로 설정한다.
5. 존을 하나씩 고르고 왼쪽 **Sliders** → 그 존의 슬라이더(존 1은 A, 존 2는 B …) Destination을 **CONTROLLERS › Volume**으로, 나머지 슬라이더는 **OFF**로.
6. **Write** → 이름 `REAPER Mixer` → 비어 있는 번호에 저장. 128번에는 덮어쓰지 않는다. 126 Internal Voices도 절대 덮어쓰지 않는다.
7. 에디터를 끈다.

셋업을 불렀을 때 슬라이더를 건드리는 순간 볼륨이 툭 바뀌는 게 싫으면, PC3X **Master → MAIN → SetupCtls**를 **Pass Entry**로 바꾼다.

### 셋업 없이 급하게 할 때

PC3X **Program** 모드에서 **Chan/Layer**로 채널을 바꿔 가며 번호를 넣는다: 채널 1 = 18, 채널 2 = 110, 채널 3 = 79, 채널 10 = 113. 마지막에 채널 1로 돌아온다. 이 방식은 슬라이더 볼륨 조절이 안 되고, 전원을 끄면 다시 해야 한다.

## 매번 시작할 때 체크리스트

설치가 끝난 뒤에는 작업할 때마다 이 순서만 지키면 된다. 순서가 바뀌면 연결이 안 잡힐 수 있다.

- [ ] PC3X 전원을 켜고 USB 연결.
- [ ] SoundTower 에디터가 켜져 있으면 끈다.
- [ ] PC3X **Setup** 모드 → `REAPER Mixer` 셋업 번호 → Enter.
- [ ] REAPER를 켠다.
- [ ] reapy 서버 실행: 자동 실행(`__startup.lua`)을 안 만들었다면 Actions → activate\_reapy\_server → Run. "running in background" 창이 뜨면 Cancel.
- [ ] Claude Desktop을 켠다. REAPER보다 나중에 켜는 게 안전하다.
- [ ] Claude Desktop에 "REAPER 프로젝트 정보 알려줘"로 연결 확인. 템포·트랙 수가 나오면 시작한다.

끝낼 때는 REAPER에서 **Ctrl + S**로 저장하고, 소리 편집이 필요하면 REAPER를 끈 뒤 SoundTower 에디터를 켠다.

## 곡 만들기

흐름은 다섯 단계다: Claude에게 .mid 요청 → REAPER에 넣기 → 트랙을 PC3X 채널로 연결 → 재생하며 다듬기 → 저장.

### 1. Claude에게 시키기 (.mid로 받기)

Claude가 REAPER에 음표를 직접 넣으면 한 번에 하나씩이라 수백 개 넣는 데 오래 걸린다. 곡 단위는 .mid 파일로 받는 게 빠르다. Claude Desktop에 아래 틀을 복사해서 원하는 부분만 바꿔 보낸다.

```
템포 90, 4/4, 8마디 곡을 4트랙짜리 .mid 파일로 만들어줘.
트랙 이름: Drums, Bass, EP, Pad
내 신디는 Kurzweil PC3X야. 음이름은 C4=60 기준.
드럼맵: 킥 C3, 스네어 A3, 닫힌 하이햇 D4, 열린 하이햇 A4, 라이드 G#5.
장르: 네오소울. 하이햇 16분, 벨로시티는 사람처럼 조금씩 다르게.
코드: Dm9 – G13 – Cmaj9 – A7(b9), 한 마디씩.
EP는 로즈 보이싱, 베이스는 E1~C3, 패드는 C3 위로 길게.
```

바꿔 볼 만한 부분: 템포, 마디 수, 장르(시티팝, 가스펠, 90년대 R&B), 코드 진행.

파일이 채팅에 뜨면 다운로드한다. 보통 `다운로드` 폴더에 저장된다.

### 2. REAPER에 넣기

1. 이전 시도에서 남은 트랙이 있으면 지운다: 트랙 이름 왼쪽 빈 곳 클릭 → Shift + 마지막 트랙 클릭 → Delete.
2. **Home** 키를 눌러 재생 위치를 맨 앞(1.1)으로 보낸다. 안 하면 파일이 엉뚱한 마디부터 들어간다.
3. 다운로드 폴더의 .mid 파일을 REAPER 트랙 영역(회색 빈 공간)으로 끌어다 놓는다.
4. 창이 뜨면 **Expand MIDI tracks to new REAPER tracks**와 **Merge MIDI tempo map**에 체크 → OK.
5. 트랙 4개가 음표째로 생긴다. 상자를 더블클릭하면 피아노 롤에 음표가 보인다.

음표 상자가 1마디가 아닌 곳에서 시작하면: 트랙 영역 클릭 → **Ctrl + A** → 아무 상자나 잡고 1.1까지 끈다.

### 3. 트랙을 PC3X 채널로 연결 (라우팅)

.mid 안의 음표가 모두 채널 1로 들어 있을 수 있어서, 트랙마다 보낼 채널을 직접 정한다. 이걸 안 하면 소리가 안 나거나 모두 같은 악기로 난다.

1. 화면 아래 믹서에서 그 트랙 세로 줄의 \*\*초록·노랑 줄무늬 버튼(ROUTE)\*\*을 누른다. 트랙 이름 오른쪽 클릭 → Routing…도 같다.
2. 창 맨 아래 **MIDI Hardware Output** → **PC3**.
3. 옆 채널 칸에서 번호를 고른다.

| 트랙 | 채널 | PC3X 소리 (REAPER Mixer 셋업) |
| --- | --- | --- |
| Drums | 10 | 113 NYC Kits |
| Bass | 2 | 110 Jaco Fretless |
| EP | 1 | 18 Stevie's Rhds |
| Pad | 3 | 79 Airy Pad |

매번 하기 귀찮으면 한 번 세팅한 뒤 **File → Save project as template**로 저장해 두고, 다음엔 템플릿을 연 상태에서 .mid의 음표만 각 트랙으로 옮긴다.

### 4. 재생하고 다듬기

- **Home → Space**로 재생한다.
- 반복 재생: 위쪽 시간 눈금을 끌어 구간 선택 → 하단 반복 버튼(↻) 켜기 → Space.
- 볼륨: PC3X **슬라이더 A\~D**로 드럼·베이스·로즈·패드를 조절한다. REAPER 믹서 페이더는 소리에 영향이 없다.
- 작은 수정은 Claude에게 말로 시킨다. 예: "Drums 트랙 하이햇 벨로시티 20 낮춰줘", "EP 트랙 볼륨 -3dB", "Bass 5\~8마디 음 수 줄여줘".
- 큰 수정(코드 진행, 장르, 마디 수)은 ".mid로 다시 줘" → 해당 트랙 상자를 지우고 새 파일을 넣는다.

### 5. 내 연주 녹음하기

1. **Ctrl + T**로 새 트랙 → 이름 Solo.
2. 트랙의 빨간 동그라미(녹음 대기)를 오른쪽 클릭 → **Input: MIDI → PC3 → All channels**.
3. ROUTE → MIDI Hardware Output → **PC3**, 채널 **4**.
4. PC3X에서 연주할 소리를 채널 4에 건다. REAPER Mixer 셋업에 존 5를 추가하는 방법: 프로그램(예: 125 Real Vibes), Channel 4, 건반 A0\~C8, Active, 슬라이더 E = Volume.
5. **Home → Ctrl + R**(녹음) → 반주 들으며 연주 → **Space**로 정지.
6. 소리가 두 번 겹쳐 나면 Solo 트랙의 모니터(스피커 아이콘)를 끈다.
7. 박자 정리는 Claude에게 "Solo 트랙 1/16 퀀타이즈 50%"처럼 시킨다.

### 6. 저장

**Ctrl + S** → 이름(예: neosoul1). 트랙, 음표, 라우팅이 같이 저장된다.

mp3·wav로 뽑으려면 PC3X 뒷면 **Main Out → 오디오 인터페이스 → 컴퓨터**로 소리를 받아 REAPER 오디오 트랙에 녹음한 뒤 File → Render한다. USB로는 소리가 안 넘어가서, 인터페이스 없이 Render하면 무음 파일이 나온다.

## 문제 해결표

실제로 세팅하면서 만난 오류와 해결법이다. 위에서부터 차례로 확인한다.

### 설치·연결 단계

| 증상 / 메시지 | 원인 | 해결 |
| --- | --- | --- |
| `python`을 인식할 수 없음 | 설치 때 PATH 체크 누락 | Python 설치 파일 다시 실행 → Modify → PATH 추가 |
| ReaScript에 "No compatible version of Python was found" | dll 폴더 칸에 파일 이름까지 넣었거나 dll 이름 오타 | 폴더 칸은 `...\Python314`까지만, dll 칸은 `python314.dll` |
| Load ReaScript 파일 형식에 `*.py`가 없음 / "No supported script files could be loaded" | **Enable Python for use with ReaScript** 체크가 풀림 | 체크 → Apply → REAPER 재시작 |
| 스크립트 실행 시 `KeyError: 'reaper'` | reapy의 자동 설정(configure\_reaper, enable\_dist\_api)이 이 PC의 REAPER 설정 파일과 안 맞음 | 자동 설정은 쓰지 않는다. 설치 5·6단계(웹 인터페이스 + 스크립트 수동 등록)로 한다 |
| Actions 목록에 activate\_reapy\_server가 없음 | 등록이 안 됨 | 설치 6단계: `C:\reapy`로 복사 후 Load ReaScript |
| cmd 테스트에 "Can't reach distant API" + `EnumProjects` 오류 | reapy 서버가 안 켜짐 | REAPER에서 activate\_reapy\_server → Run, REAPER가 켜져 있는지 확인 |
| "running in background" 창 | 서버가 이미 실행 중 | **Cancel**. Terminate는 서버를 끈다 |
| Claude Desktop에 reaper 도구가 안 보임 | config 경로 오타, `\\` 누락, 쉼표·괄호 오류, 재시작 안 함 | config 확인 → 트레이에서 Quit → 재실행. Settings → Developer에서 reaper 옆 failed 로그 확인 |
| Claude가 "REAPER에 연결 못 함" | REAPER나 reapy 서버가 꺼짐 | 체크리스트 순서대로 다시 켜기 |
| Claude가 새 프로젝트·박자 설정 실패 (`time_signature` 오류) | reaper-mcp 버그 | 무시해도 된다. 열린 프로젝트에 작업하고 박자는 REAPER 기본값 4/4 |

### 소리 단계

| 증상 | 확인 |
| --- | --- |
| 재생해도 아무 소리 없음 | Ctrl + P → MIDI Outputs에서 PC3 Enable, 트랙 ROUTE의 PC3 지정, PC3X 볼륨, SoundTower 에디터 꺼짐 |
| 일부 트랙만 소리 없음 | 그 트랙 ROUTE의 PC3·채널 번호 |
| 드럼이 피아노 소리로 남 | Drums 트랙 채널 10, PC3X 채널 10에 113 (REAPER Mixer 셋업이 켜져 있는지) |
| 모든 트랙이 같은 악기로 남 | 트랙별 채널 번호가 서로 다른지 |
| 드럼 소리가 엉뚱한 타악기 | Claude가 GM 드럼맵으로 만듦 → PC3X 드럼맵을 다시 알려주거나, 채널 10 프로그램을 951 GM Standard Kit으로 |
| 음표가 6마디쯤부터 시작 | .mid를 넣을 때 재생 위치가 거기 있었음 → Ctrl + A 후 1.1로 끌기 |
| 템포가 이상함 | .mid 넣을 때 Merge MIDI tempo map 체크, 화면 아래 템포 값 |
| PC3X 슬라이더로 볼륨이 안 바뀜 | REAPER Mixer 셋업이 켜져 있는지, 존 Status Active, 슬라이더 Destination Volume, 존 채널 = 트랙 채널 |
| 녹음할 때 소리가 두 번 겹침 | 녹음 트랙 모니터(스피커 아이콘) 끄기 |
| SoundTower 에디터가 PC3X를 못 찾음 | REAPER가 USB 포트를 잡고 있음 → REAPER 끄기 |

## 치트시트

### 경로

| 무엇 | 경로 |
| --- | --- |
| Python 폴더 | `C:\Users\SEON\AppData\Local\Programs\Python\Python314` |
| Python dll | `python314.dll` |
| MCP 서버 | `C:\Users\SEON\AppData\Local\Programs\Python\Python314\Scripts\reaper-mcp-server.exe` |
| reapy 서버 스크립트 | `C:\reapy\activate_reapy_server.py` |
| Claude 설정 파일 | Claude Desktop → Settings → Developer → Edit Config |
| REAPER 시작 스크립트 | REAPER Options → Show REAPER resource path → `Scripts\__startup.lua` |

### 번호

| 무엇 | 값 |
| --- | --- |
| reapy 웹 인터페이스 포트 | 2307 |
| reapy 서버 포트 | 2306 |
| 채널 1 / 2 / 3 / 10 | EP 18 / Bass 110 / Pad 79 / Drums 113 |
| PC3X 슬라이더 A / B / C / D | Drums / Bass / EP / Pad 볼륨 |
| PC3X 드럼맵 (C4 = 60) | 킥 C3, 스네어 A3·C4, 닫힌 하이햇 D4\~E4, 열린 하이햇 A4, 라이드 G#5, 박수 D6 |

### 확인 명령

```
python --version
where reaper-mcp-server
python -c "import reapy; print('연결 성공:', reapy.Project().n_tracks, '트랙')"
```

### REAPER 단축키

| 키 | 동작 |
| --- | --- |
| Ctrl + P | Preferences |
| Home | 맨 앞으로 |
| Space | 재생 / 정지 |
| Ctrl + R | 녹음 |
| Ctrl + T | 새 트랙 |
| Ctrl + A | 전체 선택 |
| Ctrl + S | 저장 |
| Ctrl + Z | 되돌리기 |

### 참고 링크

- [reaper-mcp (bonfire-systems)](https://github.com/bonfire-systems/reaper-mcp)
- [REAPER 다운로드](https://www.reaper.fm/)
- [SoundTower PC3 에디터 다운로드](https://www.soundtower.com/pc3/pc3_downloads.html)
