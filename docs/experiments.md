# macOS 책임 프로세스·첫 실행 실험

2026-10-16 mac-c에서 수행한 **조사용 실험**이다. 제품 구현·운영 herdr 서버 재기동·기존 pane 생성·개발자 도구 등록·키체인 unlock·launchd 설정 변경은 하지 않았다. 등록이 필요한 후속 검증은 여기서 멈췄다.

## 조건과 측정법

- macOS 27.0.1, build 26A434, arm64. 전원은 94W로 관측됐고 어댑터 표기는 96W다. 같은 머신에서 다른 작업이 실행 중이므로 처리량 benchmark로 해석하지 않는다.
- 실험 준비·컴파일·측정 script와 launcher는 `nice -n 15`로 호출했다. LaunchServices가 별도로 기동한 GUI 앱의 실제 scheduling priority는 측정하지 않았다. `/Library/Developer/CommandLineTools/usr/bin/clang`과 명시한 MacOSX.sdk로 현재 C 소스를 새로 빌드했다. 기존 binary·빌드 캐시·kache shim을 쓰지 않았다.
- 조건마다 n=6개의 **서로 다른 난수 salt를 가진 정상 C executable**을 만들었다. 10조건, 총 60개다. 모두 `codesign --verify --strict` 통과를 확인한 후 실행했다. wrapper/probe의 첫 실행 시간은 표본 시간에 포함하지 않는다.
- 각 파일의 첫 `posix_spawn+waitpid`와 즉시 이어지는 같은 파일의 두 번째 실행을 `CLOCK_MONOTONIC`으로 측정했다. 아래 첫 실행·warm·차이는 산술평균, 중앙값은 표본별 차이의 중앙값이다. 지연은 **첫 실행 − warm**이다.
- 측정 프로세스의 PID·PPID·SID, `responsibility_get_pid_responsible_for_pid`와 proc_pidpath, TCC DeveloperTool preflight raw 값, `SecKeychainGetStatus`를 함께 기록했다. preflight는 비공개 API의 참고값일 뿐이고 빠른 시간만으로 면제 여부를 단정하지 않는다.
- 키체인 상태 확인은 기본 키체인의 잠금 bit만 읽었다. 실제 agent 인증 항목의 ACL·키 접근 성공을 입증한 것은 아니다.
- baseline은 현재 WezTerm 책임 계보다. disclaim은 `responsibility_spawnattrs_setdisclaim(&attr, bool)`을 `dlsym`으로 찾아 자식 spawn에 설정했다. 이 API는 책임을 **버리는** 플래그이며 임의의 기존 GUI PID를 새 책임자로 지정하는 API가 아니다. [LLVM의 실제 사용](https://github.com/llvm/llvm-project/blob/main/lldb/source/Host/macosx/objcxx/PosixSpawnResponsible.h)도 참고했다.
- GUI 실험은 research 아래 `AAResearchHost.app`를 현재 소스로 빌드·adhoc 서명하고 LaunchServices의 `open -n -W`로 실행했다. UI 창이 없는 LSUIElement 앱이며 자기 자식 측정이 끝나면 모두 종료했다. 권한 등록·login item 등록은 하지 않았다.
- SSH 비교는 `mac-b → mac-c` 경로다. mac-c self-SSH는 host key verification 단계에서 실패했으므로 known_hosts를 바꾸지 않고, 이미 인증된 mac-b 경로를 사용했다. 실패한 호출은 성능 표본에 포함되지 않는다. SSH handshake 시간은 측정 대상이 아니다.

원본: `research/experiments/spawn_probe.c`, `run_spawn_probe.py`, `run_ssh_probe.py`, `spawn-results.json`, `ssh-results.json`, `fixed-path-signatures.json`. 소스와 로그는 외부 참고 자료와 함께 git에서 제외했고 이 문서에 수치를 보존했다.

## 실측

| 조건(n=6) | 첫 실행 평균 ms | warm 평균 ms | 차이 평균 ms | 차이 중앙값 ms | 책임자 | 키체인 |
|---|---:|---:|---:|---:|---|---|
| inherit | 1.962 | 1.823 | 0.138 | 0.133 | WezTerm | unlocked |
| disclaim0 | 1.796 | 1.687 | 0.108 | 0.069 | WezTerm | unlocked |
| setsid | 1.749 | 1.638 | 0.111 | 0.083 | WezTerm | unlocked |
| disclaim1 | 225.950 | 1.966 | 223.984 | 221.744 | 미등록 probe 자신 | unlocked |
| disclaim1-setsid | 227.895 | 1.958 | 225.938 | 225.926 | 미등록 probe 자신 | unlocked |
| stable-v1 | 2.237 | 2.125 | 0.112 | 0.105 | WezTerm | unlocked |
| stable-v2 | 2.062 | 1.899 | 0.163 | 0.182 | WezTerm | unlocked |
| gui-app | 227.043 | 5.434 | 221.609 | 219.305 | AAResearchHost.app | unlocked |
| ssh-direct | 1.766 | 1.677 | 0.090 | 0.108 | sshd-session | locked |
| ssh-gui-app | 241.976 | 5.696 | 236.280 | 218.734 | AAResearchHost.app | unlocked |

측정 완료: 15:32:10 KST(로컬 8조건), 15:34:33 KST(SSH 2조건). C probe source SHA256: `4f4ebaba002f4c216384d1906ef5fb8dbbeb31914c059fda79a0ce49a1d55045`.


## 무엇을 확인했고 무엇을 확인하지 못했는가

1. `setsid`로 SID가 자기 PID가 되어도 WezTerm 책임 프로세스는 유지됐고 +0.111ms였다. detached 판정과 TCC 책임은 같은 변수가 아니다.
2. `disclaim=false`도 책임자를 새로 지정하지 않는다. `disclaim=true`는 자식을 자기 책임자로 바꿨으며 미등록 probe에서는 +223.984ms, setsid를 추가해도 +225.938ms였다. **이 API만으로 속도 문제를 고칠 수 없다.** 등록된 고정 helper로 경계를 옮기는 변형은 가능성만 있고 미검증이다.
3. 미등록 GUI 앱은 자기 책임자가 되었고 키체인 unlocked였다. SSH에서 LaunchServices로 앱을 띄워도 같은 형태를 얻었으나 +236.280ms였다. **GUI 앱 모양만 만든다고 빨라지지 않는다.** 이미 GUI 로그인한 현재 상태에서의 결과이며 재부팅 직후 검증은 아니다.
4. 기존 등록된 sshd-session 아래는 +0.090ms로 빠르지만 키체인은 locked였다. 10-16 운영 사건처럼 속도와 키체인을 따로 합격시켜야 한다.
5. 고정 helper 실험은 helper 바이트를 그대로 두고 새 자식 바이너리 두 묶음을 실행했으며 +0.112/+0.163ms였다. 두 경우의 책임자는 **helper가 아니라 WezTerm**이었다. 이 수치는 helper 자체의 등록 유지나 실제 서버 업그레이드의 증명이 아니다.
6. 동일 절대 경로에 작은 C executable을 다시 빌드하자 adhoc 서명의 designated requirement가 아래처럼 달라졌다. **고정 경로만으로 서명 정체성까지 고정되지는 않는다.** 실제 등록된 항목을 바꾸지는 않았으므로 TCC 등록이 사라지는 사건 자체를 재현한 것은 아니다.

```text
fixed-path-identity v1: designated => cdhash H"69b440210bf80748c84a0ec58d411af9f7d062ef"
fixed-path-identity v2: designated => cdhash H"8b5c2ea32991febac3b214bf8084ec2a30cfaa34"
두 버전 모두 Signature=adhoc, TeamIdentifier=not set
```

[Apple TN2206](https://developer.apple.com/library/archive/technotes/tn2206/_index.html)은 code identity/designated requirement와 개별 보안 subsystem의 정책을 구분한다. 이를 Developer Tools의 모든 업데이트 동작을 보장하는 근거로 확대하지 않는다.

## 사용자 합격 기준별 판정

`실측`은 이번 조건에서만 확인, `가설`은 등록·업데이트·재부팅 검증 필요, `탈락`은 기준에 반한다는 뜻이다. 한 셀의 실측 합격이 후보 전체의 합격은 아니다.

| 해결 후보 | 설치 때 한 번만 등록 | 업그레이드 뒤 유지 | 재부팅·ssh/웹 우선에도 빠름 | 키체인 잠김 없음 | 지연 실측·종합 |
|---|---|---|---|---|---|
| 기존 WezTerm 계보에 의존 | 기존 등록으로 가능 | 현재 child 교체는 빠름 | GUI 터미널 기동·생존에 의존 → 탈락 | 현재만 unlocked, 순서 의존 | +0.138ms, **탈락** |
| sshd 등록 아래서 서버 기동 | 현재 sshd는 등록됨 | 새 aa 정체성의 보장 아님 | 이번 SSH는 빠름; 재부팅 미측정 | 이번 SSH는 locked → 탈락 | +0.090ms, **탈락** |
| setsid만 적용 | 면제를 주지 않음 | 책임자에 의존 | 잘못된 책임자 경로를 고치지 못함 | 로그인 문맥을 바꾸지 않음 | WezTerm 아래 +0.111ms, disclaim 아래 +225.938ms; **단독 해결 탈락** |
| responsibility disclaim API만 사용 | 미등록 자식으로 경계 이동 | 매 교체 binary 정체성 문제 | 현재부터 느림 → 탈락 | GUI 보안 세션을 만들어 주지 않음 | +223.984ms, **탈락** |
| 고정 경로의 교체 가능한 CLI launcher | adhoc 재빌드 때 DR 변경 확인 | 경로만 고정은 부족 → 탈락 | SSH Background를 고치지 못함 | locked 위험 | helper 상속 표본 +0.112/+0.163ms는 면제 증명 아님; **단독 해결 탈락** |
| 등록된 GUI 앱 → 일반 server → pane | 일회 앱 등록 **가설** | 앱 identity 유지 **가설** | LS/app 자동기동 및 server의 책임 보존 **가설** | 현재 미등록 앱은 unlocked | 미등록 앱 +221.609ms, SSH→앱 +236.280ms; **미합격** |
| **고정 AAHost.app가 pane/session host를 직접 생성** | 앱에 한 번 등록 **가설** | helper 바이트·경로·서명 고정, core만 교체 **가설** | login item/LS singleton이 유일 기동 주체 **가설** | GUI 로그인 후 LS 실측 unlocked, 재부팅 미측정 | 같은 미등록 GUI 표본만 존재; **추천 검증 후보, 아직 미합격** |
| 단순 launchd daemon/agent | 등록·launch 세션 별도 검증 필요 | helper identity에 의존 | 기존 기록에 detached/bridge 경합 | Aqua/Background 차이 남음 | 이번 설정 불변으로 미측정; **추천하지 않음** |

현재 **모든 기준을 실증한 후보는 없다**. 설치 시 등록이 필요한 실험을 현재 권한으로 넘겨서 진행하거나, 미등록 상태의 느린 결과를 성공으로 바꾸지 않았다. 다음 단계는 아래 gate를 승인받아 수행하는 것이다. 여기서 사용자에게 설정 변경을 요청하지 않고 기획서 검토를 기다린다.

## 다음 검증 gate

추천 검증 대상은 `/Applications/AAHost.app`의 고정 실행 broker다. 이 경로는 **설계 제안이며 현재 설치되지 않았다**. Developer Tools에는 이 앱을 설치 시 한 번만 등록하고, 자동기동은 설치 절차에서 SMAppService login item 승인을 한 번 받는 안이다. [Apple SMAppService](https://developer.apple.com/documentation/servicemanagement/smappservice)는 helper·login item 등록 API이지 TCC 면제를 자동 부여하는 API가 아니다.

broker는 작고 고정된 signed executable로 두고 버전별 Rust core는 앱 바깥에 둔다. 업데이트가 봉인된 앱의 내용·경로·서명을 바꾸지 않게 한다. broker가 GUI 보안 세션에서 session host/PTY 자식을 생성하고, 중심 daemon·웹·SSH bridge는 broker로 요청만 보낸다. private disclaim API나 임의 부모 PID 변경을 정상 경로에 쓰지 않는다.

| 검증 묶음 | 방법·증거 | 성공 기준 |
|---|---|---|
| 최초 등록 | 한 번 등록 후 새 salt binary 30개; responsible·signature·키체인 상태 기록 | median extra ≤10ms, p95 extra ≤20ms, 100ms 초과 표본 0; 실제 키체인 인증 항목 접근 성공 |
| core 업그레이드 | helper SHA256·DR을 전후 비교, 다른 core 실행 후 새 30개 측정 | helper 바이트·경로·서명 동일, 재등록 0, 위 지연 기준 유지 |
| 기동 경합 | CLI·웹·SSH를 동시에 먼저 연결, GUI 터미널 없이 broker 시작; singleton lock 검증 | 한 broker만 기동, fallback daemon 0, 같은 책임자·키체인·속도 |
| 재부팅 | 별도 승인된 시험 Mac에서 재부팅 → GUI 터미널 없이 ssh-first/web-first/동시 접속 | 빠른 첫 실행·키체인 접근, 순서별 30개 표본 기준 유지 |
| 로그인·잠금 | GUI 로그인 후·화면 잠금·잠자기 복귀·로그아웃·키체인 실제 잠금 구분 | 지원한다고 정한 상태에서 실제 인증 성공; 미지원 상태를 성공으로 표시하지 않음 |
| helper 장애 | 시험용 broker PID만 확인 후 종료·복귀, 기존 session 책임자 확인 | 책임 계보 소실을 탐지; 잘못된 계보에서 새 실행을 시작하지 않음; 자동 복구 가능 범위 실증 |
| helper 자체 보안 업데이트 | 동일 Developer ID/DR로 갱신이 필요할 때 별도 시험 | 재등록 없이 유지되는지 검증 전 배포 금지; immutable 정책 예외를 숨기지 않음 |

위 지연 기준은 **앞으로 적용할 제안 기준**이며 기존 probe의 OK 임계값 100ms를 낮춰 과거 결과를 통과시킨 것이 아니다. 시스템 부하가 큰 표본도 버리지 않고 원시값과 함께 설명한다. 실패하면 원인을 조사하며 반복 실행·timeout 연장으로 녹색 결과만 고르지 않는다.

재부팅 직후가 FileVault 해제·GUI 로그인 **이전**까지 뜻한다면 사용자 로그인 키체인이 자동으로 풀려 있다고 보장할 근거가 없다. 암호 저장·자동 unlock·FileVault 해제 우회는 제안하지 않는다. 이 해석에서는 현재 후보가 요구를 충족하지 못하며 요구를 몰래 완화하지 않는다. GUI 로그인 **이후**이되 WezTerm을 열지 않은 상태까지는 위 app broker의 필수 검증 대상이다. 사용자가 범위를 정하기 전에는 전자를 포함한 완전 합격으로 표시하지 않는다.

## 표본별 지연

| 조건 | 표본 1 | 2 | 3 | 4 | 5 | 6 |
|---|---:|---:|---:|---:|---:|---:|
| inherit | 0.452 | -0.114 | -0.035 | 0.043 | 0.222 | 0.262 |
| disclaim0 | 0.388 | 0.030 | 0.107 | 0.013 | -0.105 | 0.218 |
| setsid | 0.471 | 0.093 | 0.044 | 0.290 | 0.074 | -0.303 |
| disclaim1 | 221.969 | 229.323 | 220.033 | 221.519 | 229.717 | 221.341 |
| disclaim1-setsid | 232.285 | 221.783 | 222.921 | 225.271 | 226.582 | 226.783 |
| stable-v1 | 0.414 | 0.037 | 0.019 | 0.173 | -0.190 | 0.220 |
| stable-v2 | 0.373 | -0.030 | -0.012 | 0.215 | 0.149 | 0.283 |
| gui-app | 216.396 | 218.656 | 219.953 | 217.985 | 222.849 | 233.816 |
| ssh-direct | 0.348 | 0.144 | 0.073 | -0.101 | 0.336 | -0.262 |
| ssh-gui-app | 321.277 | 219.190 | 213.691 | 216.202 | 218.278 | 229.042 |

단위 ms. 표본별 첫 실행·warm·target SHA256 및 PID·SID는 로컬 원본 JSON에 있다. 실험 child와 GUI 앱은 모두 자연 종료했고 마지막 `ps -axo pid,ppid,command` 확인에서 잔여 실험 프로세스는 없었다.
