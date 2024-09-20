# <p align="center">[우리FISA 3기 클라우드 엔지니어링] crontab을 활용한 컴퓨터 자동 전원 관리 시스템 (PC-off) 🕥

---


## 개발팀원👨‍👨‍👧💻

| <img src="https://avatars.githubusercontent.com/u/65991884?v=4" width="80"> | <img src="https://avatars.githubusercontent.com/u/90691610?v=4" width="80"> | <img src="https://avatars.githubusercontent.com/u/79312705?v=4" width="80"> |
|:---:|:---:|:---:|
| [류채현](https://github.com/RyuChaeHyun) | [이석철](https://github.com/SeokCheol-Lee) | [김상민](https://github.com/isshomin) |


---
<br>

## 개요

이 프로젝트는 **crontab**을 이용하여 컴퓨터의 전원을 업무 시간에만 자동으로 켜고, 그 외의 시간에는 자동으로 종료하여 접속이 불가능하도록 설정하는 것을 목표로 한다. 이를 통해 에너지 효율성을 높이고 보안을 강화하며, 시스템의 수명을 연장할 수 있다.

정해진 근로시간에 PC를 종료시켜 정시 퇴근을 유도하고, 효율적인 근로시간 관리를 통해 직원들의 워라밸(Work and Life Balance)을 지켜주는 주 52시간 근무제 관리에 도움이 된다.

---
<br>

## 프로젝트 배경

많은 기업과 기관에서 컴퓨터가 24시간 내내 켜져 있어 불필요한 에너지 소비와 보안 위험에 노출되는 경우가 많다. 이러한 문제를 해결하고자 다음과 같은 필요성이 제기되었다.

- **에너지 절약**: 업무 시간 외에 컴퓨터를 종료함으로써 전력 소비를 감소시키고, 이에 따른 비용 절감 효과를 얻을 수 있다.
- **보안 강화**: 비업무 시간에 시스템 접근을 차단하여 외부 공격이나 내부 정보 유출의 위험을 최소화한다.
- **시스템 유지보수 효율화**: 하드웨어의 불필요한 가동을 줄여 장비의 수명을 연장하고 유지보수 비용을 절감한다.

## 주요 기능

- **자동 해제**: 지정된 업무 시작 시간에 계정잠금이 자동으로 해제된다.
- **자동 종료**: 업무 종료 시간에 컴퓨터가 자동으로 꺼진다.
- **접근 제한**: 업무 시간 외에는 원격 및 물리적 접근이 제한되어 보안성을 높인다.

## 설정 방법

### 1. 특정 시간 이후 PC 종료 설정

**Crontab 편집**

```bash
bash

crontab -e
```

다음 내용을 추가하여 주중 오후 5시 5분에 PC가 종료되도록 설정한다.

```bash
bash

5 17 * * 1-5 sudo /sbin/shutdown -h now  # 주중, 17:05에 PC 종료
```

**Shutdown 명령어에 대한 권한 부여**

`shutdown` 명령어를 비밀번호 없이 실행할 수 있도록 `sudoers` 파일을 수정한다.

```bash
bash

sudo visudo
```

다음 줄을 추가한다 (`username`을 실제 사용자 이름으로 변경).

```bash
bash

username ALL=(ALL) NOPASSWD: /sbin/shutdown
```

**Cron 상태 확인 및 시작**

```bash
bash

systemctl status cron

sudo systemctl start cron  # 실행 중이 아니면 명령어 실행

sudo systemctl enable cron  # cron 재부팅 후 자동 시작 설정

date  # 시스템 시간 확인

```

### 2. PC 종료 이후 다음 날까지 사용자 로그인 불가 설정

### PAM 시간 제한 모듈 설치

```bash
bash

sudo apt-get install libpam-time

```

### PAM 설정 파일 편집

SSH 접근을 제한하기 위해 PAM 설정 파일을 편집한다.

```bash
bash

sudo vi /etc/pam.d/sshd

```

파일의 상단에 다음 줄을 추가합니다:

```bash
bash

account    required    pam_time.so

```

### 시간 제한 규칙 설정

```bash
bash

sudo vi /etc/security/time.conf

```

파일의 끝에 다음 줄을 추가한다 (`username`을 실제 사용자 이름으로 변경):

```bash
bash

*;*;username;!Al1705-2400  # 17:05부터 다음 날 00:00까지 접근 제한

```

### 변경사항 적용

```bash
bash

sudo systemctl restart sshd

```

**주의**: 이 설정 후 해당 시간대에 해당 사용자는 접근이 불가능하다.

### 3. 스크립트를 통한 자동화 설정

업무 시간 외에는 계정을 잠그고, 업무 시작 시 계정을 잠금 해제하는 스크립트를 작성하여 자동화한다.

### evening_routine.sh 생성

```bash
# 백업 수행
backup_dir="/home/username/backups"
source_dir="/home/username"  # 백업할 디렉토리 지정
sudo rsync -avz --delete $source_dir $backup_dir

# 계정 잠금
sudo usermod -L username  # 'username'을 실제 사용자 이름으로 변경

# 시스템 종료
sudo shutdown -h now

```

### morning_routine.sh 생성

```bash
# 백업에서 복원
backup_dir="/home/username/backups"
restore_dir="/home/username"  # 복원할 디렉토리 지정
sudo rsync -avz --delete $backup_dir/ $restore_dir

# 계정 잠금 해제
sudo usermod -U username  # 'username'을 실제 사용자 이름으로 변경

```

### setup_cron.sh 생성

```bash
# 17시 작업 설정
(crontab -l ; echo "0 17 * * * /home/username/evening_routine.sh") | crontab -

# 08시 작업 설정
(crontab -l ; echo "0 8 * * * /home/username/morning_routine.sh") | crontab -

# root 권한으로 실행되어야 하는 작업을 위한 설정
echo "0 17 * * * /home/username/evening_routine.sh" | sudo tee -a /etc/crontab
echo "0 8 * * * /home/username/morning_routine.sh" | sudo tee -a /etc/crontab

```

### 스크립트에 실행 권한 부여

```bash
chmod +x evening_routine.sh morning_routine.sh setup_cron.sh

41 8 * * * DISPLAY=:0 XAUTHORITY=/home/username/.Xauthority notify-send "저장하세요!" "지금은 17시입니다. 파일을 저장하세요!"
42 8 * * * DISPLAY=:0 XAUTHORITY=/home/username/.Xauthority notify-send "저장하세요!" "지금은 17시 55분입니다. 파일을 저장하세요!"
```

### 스크립트 실행

```bash
./setup_cron.sh
```

## 기대 효과

- **운영 비용 절감**: 전력 소비 감소로 인한 비용 절감 효과
- **보안 강화**: 업무 시간 외 시스템 접근 제한으로 보안 위험 감소
- **환경 보호**: 에너지 절약을 통한 탄소 배출량 감소로 환경에 기여

## 아쉬웠던 점

1. **자동 부팅 구현의 제약**: 자동으로 출근시간이 되면 자동 부팅이 되게 구현하고 싶었으나,  VirtualBox 내에서 BIOS 설정을 변경하지 않고 자동으로 시스템을 부팅하는 기능을 구현하는 데 한계가 있었다. 이를 해결하기 위해 다른 하드웨어 접근 방식이나 추가적인 소프트웨어 설정이 필요했다. 
2. **알림 기능의 한계**: 업무 종료 알림을 제공하는 기능이 원활하게 작동하지 않아 사용자에게 적시에 경고를 주는 데 어려움이 있었다.
