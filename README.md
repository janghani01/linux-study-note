# 🐧 Linux Administration & Troubleshooting Note

## 1. CPU 사용률이 높을 때
* `top` 명령어 확인
  * `load average`가 코어 수보다 낮은 게 정상 / 좀 높으면 주의 / 2배 이상이면 위험.
  * `load average`는 높고 `idle`이 낮으면: **CPU 과부하 상태**
  * `load average`도 높고 `idle`도 높으면: **디스크 또는 I/O 병목 문제**
* `sar` 명령어 확인
  * 로그 경로: `cd /var/log/sa` (Rocky) 또는 `cd /var/log/sysstat` (Ubuntu)
  * 파일 열기: `sar` / 파일 확인: `-f` / CPU 확인: `-u` / 메모리 확인: `-r`
  * 예시: `sar -u -f sa01` (sa01 파일의 CPU 확인)
* 지표 설명
  * `%user`: 내가 현재 쓰고 있는 CPU 사용량
  * `%idle`: CPU 남은 사용량 (CPU 사용량 계산 = 100 - %idle)
  * `%iowait`: 디스크 때문에 기다리는 시간 (20% 이상이면 위험)
  * `%steal`: 가상화 환경에서 CPU 뺏긴 시간

## 2. CPU / 메모리 사용량 높은 순 정렬
* CPU 사용량 내림차순: `ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 11`
* 메모리 사용량 내림차순: `ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%mem | head -n 11`

## 3. uptime과 who -b 차이점
* `who -b`: 시스템 마지막 부팅 시점. `/var/run/utmp` 기반으로 커널이 실제로 부팅된 시간을 기록 (로그 기반).
* `uptime`: 현재까지 시스템이 켜져 있던 시간. `/proc/uptime` 기반으로 커널이 올라온 이후 경과 시간을 계산 (계산 기반).
> **시간이 다를 경우:** 부팅 실패 후 재시도 시, Emergency mode 진입 시, utmp 기록 꼬임, 컨테이너 환경

## 4. 접속 이력 확인
* `last -n 10`: `/var/log/wtmp`에 저장된 사용자 접속 이력(마지막 접속 10개) 출력
* SSH 서버 접속 로그:
  * Rocky (`/var/log/secure`): `sudo grep "sshd" /var/log/secure`
  * Ubuntu (`/var/log/auth.log`): `sudo tail -f /var/log/auth.log | grep "Accepted"`

## 5. 통신 명령어
* 라우팅 경로/네트워크 끊김 확인: `traceroute [ip] -p [port]`
* 방화벽/포트 확인: `curl -v telnet://[ip]:[port]`
  * `Connection refused` -> 포트 닫힘 / `Connection timed out` -> 방화벽 차단 가능성
* 도메인 접속 테스트: `nslookup [도메인네임]`
* tcpdump (telnet 안될 때 등 패킷 확인):
  * IP 조회: `tcpdump -i eth0 host [ip]` / 포트 조회: `tcpdump -i eth0 port [port]`
  * 출발지: `tcpdump -i eth0 src [ip]` / 도착지: `tcpdump -i eth0 dst [ip]`
* 임의 포트 오픈: `openssl s_server -accept [포트번호] -nocert -quiet`
* 네트워크 연결 상태 (`netstat -antp`):
  * `-a` (모든 소켓), `-n` (숫자로 표시-빠름), `-t` (TCP), `-u` (UDP), `-p` (프로세스 PID)
  * 현재 접속 중인 세션 확인: `netstat -antp | grep -i established`
  * *참고: netstat은 구버전이며 ss 명령어를 더 많이 쓰는 추세임.*

## 6. 마운트 (Mount & Storage)
### 6-1. 일반 파티션 마운트 (fdisk)
1. 목록 확인: `lsblk`
2. 파티션 관리: `fdisk /dev/vdb` (n 추가 -> p 확인 -> 1 -> 엔터 -> 엔터 -> w 저장)
3. 포맷 (ext4): `mkfs.ext4 /dev/vdb1`
4. 마운트: `mkdir /data` -> `mount /dev/vdb1 /data`
5. 자동 마운트 등록: `vi /etc/fstab`에 `/dev/vdb1 /data ext4 defaults 0 0` 추가 -> `mount -a` 적용

### 6-2. 마운트 증설 (growpart)
* `lsblk` -> `sudo growpart /dev/vdb 1` (파티션 확장) -> `sudo resize2fs /dev/vdb1` (파일 시스템 확장) -> `df -h`
* *패키지 설치(Rocky):* `dnf install cloud-utils-growpart`

### 6-3. umount 설정
* `lsof /dev/vdb`로 사용 중인 프로세스 확인 후 종료 (열린 파일이 있으면 umount 안 됨)
* `umount /dev/vdb` 실행

### 6-4. GPT 파티션
1. `lsblk` -> `parted /dev/vdb`
2. `mklabel gpt` (변환) -> `mkpart primary 0% 100%` (용량 설정) -> `quit`
3. 포맷 및 마운트: `mkfs.ext4 /dev/vdb1` -> `mkdir /data` -> `mount /dev/vdb1 /data`
4. 자동 마운트: `/etc/fstab` 등록 후 `mount -a`

### 6-5. File Storage (NFS)
1. `lsblk`
2. 패키지 설치: `yum install nfs-utils` (Redhat) / `apt-get install nfs-common` (Debian)
3. RPC 데몬 활성화: `systemctl enable rpcbind.service` -> `systemctl start rpcbind.service`
4. 마운트: `mkdir /data` -> `mount -t nfs [NFS_IP]:/[경로] /home2/irteam`
5. 자동 마운트: `/etc/fstab`에 `[NFS_IP]:/[경로] /home2/irteam nfs defaults,vers=3,_netdev,noresvport 0 0` 추가

### 6-6. LVM (Logical Volume Manager)
1. 패키지 설치: `dnf install lvm2 -y` (Redhat) / `sudo apt update && sudo apt install lvm2 -y` (Debian)
2. PV/VG/LV 생성: `pvcreate /dev/vdd` -> `vgcreate datavg /dev/vdd` -> `lvcreate -l 100%FREE -n datalv datavg`
3. 포맷 및 마운트: `mkfs.ext4 /dev/datavg/datalv` -> `mkdir /data2` -> `mount /dev/datavg/datalv /data2`
4. 자동 마운트: `blkid /dev/datavg/datalv`로 UUID 확인 후 `/etc/fstab`에 `UUID=[값] /data2 ext4 defaults 0 0` 등록
> **LVM 용량 증설:** `pvresize /dev/vdb` -> `vgdisplay datavg` -> `lvextend -l +100%FREE /dev/datavg/datalv` -> `resize2fs /dev/datavg/datalv` -> `df -hT | grep data`

### 6-7. vi 편집기 팁
* 복사/붙여넣기: `Shift+v` (줄 선택) -> `y` (복사) -> `p` (붙여넣기)
* 전체 텍스트 날리기: `:%d` 엔터

## 7. 스왑 (Swap) 생성 및 증설
*(※ 스왑 용량은 보통 메모리의 2배 정도 설정)*
* **스왑 생성:**
  1. 상태 확인: `free -h` / `top` / `swapon -s`
  2. 파일 생성: `fallocate -l [용량] /swapfile`
  3. 권한 부여: `chmod 600 /swapfile`
  4. 포맷 및 활성화: `mkswap /swapfile` -> `swapon /swapfile`
  5. 자동 등록: `vi /etc/fstab`에 `/swapfile swap swap defaults 0 0` -> `mount -a`
* **스왑 증설:**
  1. 비활성화 및 삭제: `swapoff /swapfile` -> `rm -f /swapfile`
  2. 재생성: `fallocate -l 32G /swapfile` (에러 시 `dd if=/dev/zero of=/swapfile bs=1G count=32` 사용)
  3. 이후 권한 부여, mkswap, swapon 동일하게 진행

## 8. 위치 및 파일 찾기
* 현재 경로: `pwd`
* 파일 찾기: `find / -name [파일명]` 또는 `find / -type f -name [파일명] 2>/dev/null`
* 실행 파일 찾기: `which [실행파일명]`

## 9. 디렉토리 및 파일 생성
* 디렉토리 생성: `mkdir [경로] [디렉토리명]`
* 파일 생성: `echo "" > [텍스트파일명]` 또는 `touch [텍스트파일명]`
* `echo`: `>` (덮어쓰기), `>>` (추가하기, ex: `echo "PermitRootLogin no" >> /etc/ssh/sshd_config`)
* `touch`: 파일 시간 정보를 갱신하며, 파일이 없으면 생성.

## 10. 자동화 설정 (crontab)
* 정해진 시간에 명령어를 실행하는 스케줄러. 설정 전 대상 스크립트 권한 부여(`chmod 700`) 필요.
* 추가/수정: `crontab -e`
* 확인: `crontab -l` / 삭제: `crontab -r`
* 형식: `분 시 일 월 요일 실행할명령어` (ex: `0 2 * * * /home/jang/test.sh` -> 매일 새벽 2시 실행)

## 11. sysstat (SAR) 수집 주기 변경 (10분 -> 1분)
1. `sed -i 's/10/1/g' /usr/lib/systemd/system/sysstat-collect.timer` (10분 -> 1분 치환)
2. `sed -i '/SADC_OPTIONS/d' /etc/sysconfig/sysstat` (기존 문구 삭제 / 우분투는 `/etc/sysstat/sysstat`)
3. `echo 'SADC_OPTIONS="-S DISK"' >> /etc/sysconfig/sysstat` (새 문구 추가)
4. `systemctl daemon-reload` (설정 리로드)
5. `systemctl enable sysstat-collect.timer` -> `systemctl restart sysstat-collect.timer`
> **sed 명령어:** `-i`(직접 수정), `s`(치환), `g`(전체 변경), `d`(삭제)

## 12. 파일 비교 (diff)
* `diff -u 파일1 파일2` (설정 백업본과 수정본 비교 시 직관적)
  * `---` (원본), `+++` (수정본), `-` (삭제), `+` (추가)
* 옵션: `-i` (대소문자 무시), `-w` (공백/들여쓰기 무시), `-r` (디렉토리 전체 비교), `-q` (요약 보고), `-s` (같아도 출력)

## 13. 원격 서버 파일 전송 및 스크립트 실행 (scp / ssh)
* 복사해오기: `scp -i [펨키] [계정]@[ip]:/[가져올경로] .`
* 복사보내기: `scp [전송할파일] [서버]:[경로]` (포트 지정은 `ssh`는 `-p`, `scp`는 `-P`)
* 스크립트 끌어다 쓰기: `ssh -t -i [펨키] [계정]@[ip] "[실행위치]"` (사전 권한 부여 필요, 400 등 제한적 권한 주의)

## 14. DB 서버 원격 접속 (MySQL / MariaDB)
* 접속: `mysql -h 172.16.16.167 -P 3306 -u admin -p` (비밀번호는 쳐서 들어가는 것이 보안에 좋음)
* 외부 쿼리 실행: `mysql -uroot -p -e "SELECT * FROM [테이블명];"`
* 백업: `mysqldump -uroot -p scp_backup > scp_backup_dump.sql`
> 옵션: `-h`(Host), `-P`(Port), `-u`(User), `-p`(Password), `-e`(Execute), `-t`(Table 형식), `-v`(상세 모드)

## 15. 텍스트 파일 줄 개수 찾기 (wc)
* 텍스트 줄 수(`-l`), 단어 수(`-w`), 바이트 수(`-c`), 문자 수(`-m`)
* 활용:
  * 서버 파일 개수: `ls -l | grep wc -l`
  * TCP 연결 개수: `netstat -ant | grep ESTABLISHED | wc -l`

## 16. 라우트 테이블 규칙 추가 (NIC 추가 / iproute2)
1. `echo "203 relay93" >> /etc/iproute2/rt_tables`
2. `ip rule add from 172.16.0.93 table relay93`
3. `ip route add 172.16.0.0/24 dev eth4 src 172.16.0.93 table relay93`
4. 확인: `ip route get 172.16.0.3 from 172.16.0.93`

## 17. SCP NTP 설정 (chrony)
1. `touch /etc/chrony.conf`
2. `echo "server 198.19.0.54 iburst" > /etc/chrony.conf`
3. `systemctl start chronyd.service`
4. `timedatectl status`
> 추가 확인: `date`, `timedatectl`, `chronyc sources -v`, `chronyc tracking`(System time이 0에 가까워야 함), `chronyc makestep`

## 18. 리눅스 계정 생성
* 생성: `useradd [계정명]`
* 패스워드: `passwd [계정명]`
* 권한(관리자): `usermod -aG wheel [계정명]`

## 19. 기타
* 호스트네임 변경: `hostnamectl set-hostname [변경할 이름]`
* 버퍼/캐시 초기화: `sync; echo 1 > /proc/sys/vm/drop_caches; free -m` (`sync`는 메모리 데이터를 디스크에 저장)
* OS 버전 확인: `cat /etc/os-release` 또는 `hostnamectl`

---

## ☁️ 클라우드 및 트러블슈팅 사례
### 1. 망연계 서버 <-> 오브젝트 스토리지 통신 이슈
* **증상:** 망연계 서버에서 오브젝트 스토리지 Public 경로로 curl 통신 불가 (ping 정상, TCP 443 연결 정상이나 HTTPS 요청 실패).
* **원인:** MTU(Maximum Transmission Unit) 값 불일치. 서버 패킷 크기(9000)가 망연계 구간 제한(1500)에 걸려 Packet Drop 발생.
* **조치:** `sudo ip link set dev eth0 mtu 1500` 설정 후 curl 통신 성공. (`ip addr`로 확인)

### 2. Object Storage 접근 권한 수정 시 확인 사항
* (추후 작성 예정)
