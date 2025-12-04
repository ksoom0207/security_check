# Ansible Security Hardening Playbook

이 프로젝트는 리눅스 서버의 보안 강화를 위한 Ansible 플레이북입니다. 기존 Bash 스크립트를 Ansible로 변환하여 관리 편의성과 재사용성을 높였습니다.

## 프로젝트 구조

```
project_root/
├── inventory.ini             # 호스트 정의 파일
├── setup_ssh_keys.yml        # SSH 키 교환 플레이북 (최초 1회 실행)
├── harden_playbook.yml       # 메인 보안 강화 플레이북
├── group_vars/
│   └── all.yml               # 전역 변수 (배너 문구 등)
├── templates/                # 설정 파일 템플릿 (.j2)
│   ├── common-password.j2    # [U-02] PAM 패스워드 정책
│   ├── common-auth.j2        # [U-03] PAM 인증 설정
│   ├── cron.allow.j2         # [U-22] Cron 허용 사용자
│   ├── issue.j2              # [U-68] 로그인 배너
│   └── motd.j2               # [U-68] MOTD 배너
└── README.md                 # 이 파일
```

## 주요 기능

### 🔍 Pre-Check (사전 검사)
플레이북 실행 전에 자동으로 필수 파일의 존재 여부를 확인합니다:
- **자동 파일 검사**: 설정 변경 전 대상 파일이 존재하는지 확인
- **명확한 경고 메시지**: 파일이 없을 경우 어떤 보안 항목이 건너뛰어지는지 표시
- **안전한 실행**: 존재하지 않는 파일에 대한 작업을 자동으로 건너뜀
- **상세한 로그**: 각 호스트별로 어떤 파일이 있고 없는지 상세히 표시

검사 대상 파일:
- `/etc/ssh/sshd_config` (SSH 설정)
- `/etc/pam.d/common-*` (PAM 설정)
- `/etc/login.defs` (패스워드 정책)
- `/etc/profile` (세션 타임아웃)
- `/etc/shadow` (사용자 패스워드)
- `/etc/rsyslog.conf` (로그 설정)
- `/usr/bin/crontab` (Cron 바이너리)
- `/etc/apache2/apache2.conf` (Apache 설정, web_controllers만)

### 💡 조건부 실행
파일이 존재하지 않으면 해당 작업을 자동으로 건너뛰고, 명확한 경고 메시지를 표시합니다.

예시:
```
TASK [Pre-Check] Display missing critical files warning
ok: [server1] => {
    "msg": "WARNING: The following critical files are missing:\n- /etc/apache2/apache2.conf (Apache hardening will be skipped)\n"
}

TASK [U-35] Disable Indexes in apache2.conf
skipping: [server1]

TASK [Apache] Warning - Apache config not found
ok: [server1] => {
    "msg": "WARNING: /etc/apache2/apache2.conf not found. Skipping Apache hardening configuration."
}
```

### 🎯 사용자 확인 단계
Pre-check 완료 후 실제 작업 실행 전에 사용자에게 확인을 요청합니다:

```
============================================================
모든 필수 파일 검사가 완료되었습니다.

보안 설정을 적용하시겠습니까?

- Enter를 누르면 진행합니다
- Ctrl+C를 누른 뒤 'A'를 입력하면 중단합니다

자동 실행 시 --extra-vars "skip_confirmation=true" 사용
============================================================
```

### 📊 상세한 결과 리포트
작업 완료 후 화면에 상세한 리포트가 표시되고, 파일로도 저장됩니다:

**화면 출력 예시:**
```
╔══════════════════════════════════════════════════════════════╗
║        보안 강화 작업 완료 리포트 (Summary Report)           ║
╚══════════════════════════════════════════════════════════════╝

📋 호스트: server1
📅 완료 시각: 2025-12-04T10:30:45Z

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ 적용된 보안 항목
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

[계정 및 인증]
  ✓ U-01: SSH Root 로그인 제한
  ✓ U-02: 패스워드 복잡도 설정 (PAM)
  ✓ U-03: 계정 잠금 임계값 설정 (5회/120초)
  ✓ U-46: 패스워드 최소 길이 (8자)
  ✓ U-47: 패스워드 최대 사용기간 (90일)
  ...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
📊 통계 (PLAY RECAP 참조)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- ok: 성공한 작업 (변경 없음 포함)
- changed: 실제로 변경된 항목 ← 이 값으로 실제 변경 사항 확인
- skipped: 조건 불충족으로 건너뛴 항목
- failed: 실패한 작업
```

**리포트 파일:**
- 위치: `/tmp/security_hardening_report_<hostname>_<date>.txt`
- 형식: 텍스트 파일
- 내용: 적용된 보안 항목, 백업 파일 위치, 다음 단계 안내

## 적용되는 보안 항목

### 계정 관리
- **U-01**: SSH Root 로그인 제한
- **U-02**: 패스워드 복잡도 설정 (PAM)
- **U-03**: 계정 잠금 임계값 설정 (5회 실패 시 120초 잠금)
- **U-46**: 패스워드 최소 길이 설정 (8자 이상)
- **U-47**: 패스워드 최대 사용 기간 설정 (90일)
- **U-49**: 불필요한 계정 제거 (lp, uucp, games)
- **U-50**: 불필요한 그룹 제거 (lp, uucp, games)

### 파일 및 디렉토리 관리
- **U-08**: /etc/shadow 파일 권한 강화 (400)
- **U-11**: /etc/rsyslog.conf 파일 권한 강화 (640)
- **U-13**: 불필요한 SUID/SGID 제거

### 서비스 관리
- **U-22**: Cron 접근 제어
- **U-54**: 세션 타임아웃 설정 (600초)
- **U-68**: 경고 배너 설정

### Apache 웹 서버 (web_controllers 그룹만 해당)
- **U-35**: 디렉토리 목록 표시 비활성화
- **U-37**: AllowOverride 설정
- **U-39**: FollowSymLinks 비활성화
- **U-40**: /srv/ 디렉토리 접근 제한
- **U-41**: DocumentRoot 설정

## 사용 방법

### 0. SSH 키 교환 설정 (최초 1회)

보안 강화 플레이북을 실행하기 전에 먼저 SSH 키 인증을 설정해야 합니다.

#### SSH 키 자동 설정 (권장)
```bash
ansible-playbook setup_ssh_keys.yml
```

**실행 흐름:**
1. 원격 호스트의 사용자명 입력 (기본값: 현재 사용자)
2. SSH 패스워드 입력
3. sshpass 자동 설치 (없는 경우)
4. SSH 키 생성 (없는 경우)
5. 모든 호스트에 SSH 공개키 자동 복사
6. 연결 테스트

**예시:**
```bash
$ ansible-playbook setup_ssh_keys.yml

Enter remote username (default: ubuntu): ubuntu
Enter SSH password for remote hosts:

TASK [Setup] Display setup information
ok: [localhost] =>
  msg: |-
    ============================================================
    SSH 키 교환 설정 시작
    ============================================================
    대상 호스트: 3개
    원격 사용자: ubuntu
    ============================================================

...

TASK [Setup] Display connection test results
ok: [localhost] =>
  msg: |-
    ✓ 모든 호스트에 SSH 키 인증이 성공적으로 설정되었습니다!
    이제 보안 강화 플레이북을 실행할 수 있습니다
```

#### 수동 SSH 키 설정
```bash
# 1. SSH 키 생성 (없는 경우)
ssh-keygen -t rsa -b 4096

# 2. 각 호스트에 키 복사
ssh-copy-id user@192.168.1.10
ssh-copy-id user@192.168.1.11

# 3. 연결 테스트
ssh user@192.168.1.10 echo "test"
```

### 1. Inventory 설정

`inventory.ini` 파일을 편집하여 대상 서버를 정의합니다:

```ini
[all_servers]
server1 ansible_host=192.168.1.10 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa
server2 ansible_host=192.168.1.11 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_rsa

[web_controllers]
server1  # Apache 설정이 필요한 서버만 추가
```

### 2. 변수 커스터마이징

`group_vars/all.yml` 파일에서 변수를 수정할 수 있습니다:

```yaml
password_max_days: 90        # 패스워드 최대 사용 기간
password_min_len: 8          # 패스워드 최소 길이
password_min_days: 1         # 패스워드 최소 사용 기간
session_timeout: 600         # 세션 타임아웃 (초)
cron_allowed_users:          # Cron 허용 사용자
  - root
  - your_admin_user
```

### 3. 실행 방법

#### 기본 실행 (대화형 모드)
```bash
ansible-playbook -i inventory.ini harden_playbook.yml
```

**실행 흐름:**
1. **Pre-Check**: 필수 파일 존재 여부 확인
2. **사용자 확인**: Enter를 눌러 계속 진행 (또는 Ctrl+C로 중단)
3. **보안 강화 작업 실행**
4. **Summary Report**: 상세한 결과 리포트 출력
5. **리포트 파일 저장**: `/tmp/security_hardening_report_<hostname>_<date>.txt`

#### 자동 실행 (확인 단계 건너뛰기)
```bash
ansible-playbook -i inventory.ini harden_playbook.yml --extra-vars "skip_confirmation=true"
```

CI/CD 파이프라인이나 자동화된 환경에서 사용할 때 유용합니다.

#### Check Mode (실제 변경 없이 확인만)
```bash
ansible-playbook -i inventory.ini harden_playbook.yml --check --diff
```

#### 특정 태그만 실행
```bash
# SSH 설정만 적용
ansible-playbook -i inventory.ini harden_playbook.yml --tags ssh

# Apache 설정만 적용
ansible-playbook -i inventory.ini harden_playbook.yml --tags apache

# PAM 설정만 적용
ansible-playbook -i inventory.ini harden_playbook.yml --tags pam
```

#### 특정 호스트만 실행
```bash
ansible-playbook -i inventory.ini harden_playbook.yml --limit server1
```

#### 결과를 파일로 저장
```bash
ansible-playbook -i inventory.ini harden_playbook.yml > result.log 2>&1
```

### 4. 결과 해석

실행 후 PLAY RECAP이 표시됩니다:

```
PLAY RECAP *********************************************************************
server1 : ok=35   changed=12   unreachable=0    failed=0    skipped=0
server2 : ok=30   changed=8    unreachable=0    failed=0    skipped=5
```

- **ok**: 작업 성공 (변경 없음 또는 변경 완료)
- **changed**: 설정이 변경됨
- **unreachable**: 서버에 접속 불가
- **failed**: 작업 실패
- **skipped**: 조건에 맞지 않아 건너뜀

### 5. 실패 원인 분석

**Pre-Check 기능 덕분에 대부분의 파일 부재 오류는 사전에 감지되고 안전하게 건너뛰어집니다.**

실제 오류가 발생한 경우 빨간색으로 상세한 오류 메시지가 출력됩니다:

```
TASK [U-03] Install libpam-pwquality] ******************************************
fatal: [server1]: FAILED! => {
    "msg": "Failed to update apt cache: ..."
}
```

**Pre-Check를 통해 예방되는 일반적인 오류:**
- ❌ 과거: `Destination /etc/apache2/apache2.conf does not exist !` → 플레이북 중단
- ✅ 현재: `WARNING: /etc/apache2/apache2.conf not found. Skipping Apache hardening.` → 안전하게 계속 진행

**skipped 항목 확인:**
```
PLAY RECAP *********************************************************************
server1 : ok=35   changed=12   unreachable=0    failed=0    skipped=15
                                                              ↑
                                                   15개 작업이 조건에 맞지 않아 건너뜀
```

skipped가 많다면:
1. Pre-Check 경고 메시지를 확인하여 어떤 파일이 없는지 파악
2. 해당 파일/패키지를 설치할지 결정
3. 필요 없는 항목이면 무시

## 전체 워크플로우

### 📋 완전한 실행 순서

```bash
# 1단계: SSH 키 교환 설정 (최초 1회만)
ansible-playbook setup_ssh_keys.yml

# 2단계: 보안 강화 플레이북 실행
ansible-playbook -i inventory.ini harden_playbook.yml

# 3단계: 결과 확인
cat /tmp/security_hardening_report_<hostname>_<date>.txt
```

### 🔄 전체 프로세스

```
┌─────────────────────────────────────────────────────────┐
│ 1. 환경 준비                                             │
├─────────────────────────────────────────────────────────┤
│ • inventory.ini 편집 (호스트 정보)                       │
│ • group_vars/all.yml 편집 (변수 설정)                    │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 2. SSH 키 교환 (최초 1회)                                │
├─────────────────────────────────────────────────────────┤
│ ansible-playbook setup_ssh_keys.yml                     │
│                                                          │
│ • sshpass 설치                                           │
│ • SSH 키 생성                                            │
│ • 모든 호스트에 키 복사                                  │
│ • 연결 테스트                                            │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 3. 보안 강화 실행                                        │
├─────────────────────────────────────────────────────────┤
│ ansible-playbook -i inventory.ini harden_playbook.yml   │
│                                                          │
│ 3.1. Pre-Check (파일 존재 확인)                          │
│ 3.2. Confirmation (사용자 확인)                          │
│ 3.3. Security Hardening (보안 설정 적용)                 │
│ 3.4. Verification (설정 검증)                            │
│ 3.5. Summary Report (결과 리포트)                        │
└─────────────────────────────────────────────────────────┘
                          ↓
┌─────────────────────────────────────────────────────────┐
│ 4. 사후 확인                                             │
├─────────────────────────────────────────────────────────┤
│ • 리포트 파일 확인                                       │
│ • 새 세션에서 로그인 테스트                              │
│ • 서비스 정상 작동 확인                                  │
│ • 패스워드 정책 테스트                                   │
└─────────────────────────────────────────────────────────┘
```

### ⚡ 빠른 시작 (Quick Start)

완전 자동화 실행:
```bash
# 1. SSH 키 설정
ansible-playbook setup_ssh_keys.yml

# 2. 보안 강화 (확인 단계 스킵)
ansible-playbook -i inventory.ini harden_playbook.yml \
  --extra-vars "skip_confirmation=true"
```

## 주의사항

### 실행 전 확인사항
1. **백업**: 중요한 설정 파일은 자동으로 백업되지만, 전체 시스템 백업을 권장합니다
2. **테스트**: 프로덕션 환경 적용 전 테스트 환경에서 먼저 실행하세요
3. **접근 권한**: Ansible 실행 사용자가 sudo 권한을 가지고 있어야 합니다
4. **SSH 접근**: 대상 서버에 SSH로 접속 가능해야 합니다

### PAM 설정 관련
- PAM 설정 변경 후 로그인이 불가능할 수 있으므로, **현재 세션을 유지한 채로 새 세션에서 로그인 테스트**를 해야 합니다
- 문제 발생 시 백업 파일(`*_org`)로 복원할 수 있습니다

### Apache 설정 관련
- Apache가 설치되지 않은 서버는 `web_controllers` 그룹에서 제외하세요
- Apache 설정 변경 후 구문 오류가 있으면 서비스가 시작되지 않을 수 있습니다

## 고급 사용법

### 병렬 실행
```bash
# 10개 호스트를 동시에 처리
ansible-playbook -i inventory.ini harden_playbook.yml -f 10
```

### Verbose 모드
```bash
# 상세한 실행 로그 확인
ansible-playbook -i inventory.ini harden_playbook.yml -vvv
```

### 환경별 실행
```bash
# 개발 환경
ansible-playbook -i inventory_dev.ini harden_playbook.yml

# 운영 환경
ansible-playbook -i inventory_prod.ini harden_playbook.yml
```

## 검증

플레이북 실행 후 자동으로 검증 태스크가 실행됩니다:
- 패스워드 정책 확인
- 제거된 사용자/그룹 확인
- Cron 설정 확인

수동 검증:
```bash
# 특정 서버의 설정 확인
ansible -i inventory.ini server1 -m command -a "grep PASS /etc/login.defs"
ansible -i inventory.ini server1 -m command -a "cat /etc/cron.allow"
```

## 트러블슈팅

### 1. "Permission denied" 오류
```bash
# SSH 키 확인
ansible -i inventory.ini all -m ping

# sudo 권한 확인
ansible -i inventory.ini all -m command -a "sudo -l" --ask-pass
```

### 2. 패키지 설치 실패
```bash
# apt 캐시 업데이트
ansible -i inventory.ini all -m apt -a "update_cache=yes"
```

### 3. 파일이 존재하지 않음
해당 항목은 `ignore_errors: yes`가 설정되어 있어 플레이북 실행이 계속됩니다.
필요한 경우 해당 태스크를 수정하거나 제거하세요.

## 라이선스

이 프로젝트는 보안 강화 목적으로 사용됩니다.

## 기여

개선 사항이나 버그 리포트는 이슈로 등록해 주세요.
