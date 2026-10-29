# 🐧 Linux Administration & Troubleshooting Note

## 1. 시스템 리소스 및 성능 모니터링
### 1-1. CPU 사용률이 높을 때
* `top`: `load average`가 코어 수보다 낮으면 정상, 높으면 주의, 2배 이상이면 위험.
  * load average는 높으나 idle이 낮으면: **CPU 과부하**
  * load average도 높고 idle도 높으면: **디스크 또는 I/O 병목 문제**
* `sysstat` (SAR) 확인
  * 경로: `/var/log/sa` (Rocky) / `/var/log/sysstat` (Ubuntu)
  * 명령어: `sar -u -f sa01` (CPU), `sar -r -f sa01` (메모리)
  * `%user`: 현재 사용 중인 CPU 사용량 / `%idle`: 남은 CPU (100 - %idle = CPU 사용량)
  * `%iowait`: 디스크 대기 시간 (20% 이상 시 위험) / `%steal`: 가상화 환경에서 CPU를 뺏긴 시간

### 1-2. 프로세스 자원 사용량 정렬
* CPU 사용량 내림차순: `ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%cpu | head -n 11`
* 메모리 사용량 내림차순: `ps -eo pid,ppid,cmd,%cpu,%mem --sort=-%mem | head -n 11`

### 1-3. 시스템 부팅 시간 및 가동 시간
* `who -b`: 마지막 부팅 시점 출력 (`/var/run/utmp` 기반)
* `uptime`: 부팅 후 경과 시간 출력 (`/proc/uptime` 기반)
> **시간이 다를 경우:** 부팅 실패 후 재시도, Emergency mode 진입, utmp 기록 꼬임, 컨테이너 환경 등

---

## 2. 네트워크 및 통신 트러블슈팅
### 2-1. 네트워크 상태 확인 (`netstat` / `ss`)
* 현재 서버 네트워크 연결 상태: `netstat -antp`
* 현재 접속 중인 세션 확인: `netstat -antp | grep -i established`

### 2-2. 통신 테스트 및 패킷 캡처
* 라우팅 경로 확인: `traceroute [ip] -p [port]`
* 방화벽/포트 확인: `curl -v telnet://[ip]:[port]` (Connection refused = 닫힘 / timed out = 방화벽 차단)
* DNS 도메인 조회: `nslookup [도메인]`
* 패킷 캡처 (`tcpdump`):
  * `tcpdump -i eth0 host [ip]` (IP 조회)
  * `tcpdump -i eth0 port [port]` (포트 조회)
* 임의 포트 오픈: `openssl s_server -accept [포트번호] -nocert -quiet`

### 2-3. 접속 이력 확인
* 최근 접속자 10명: `last -n 10` (`/var/log/wtmp` 기반)
* SSH 접속 로그 확인:
  * Rocky: `sudo grep "sshd" /var/log/secure`
  * Ubuntu: `sudo tail -f /var/log/auth.log | grep "Accepted"`

---

## 3. 디스크 마운트 및 스토리지 관리 (LVM / NFS)
### 3-1. 일반 파티션 마운트 (fdisk / ext4)
1. 파티션 생성: `fdisk /dev/vdb` (n -> p -> 1 -> w)
2. 파일시스템 생성: `mkfs.ext4 /dev/vdb1`
3. 수동 마운트: `mount /dev/vdb1 /data`
4. 자동 마운트 등록: `vi /etc/fstab` -> `/dev/vdb1 /data ext4 defaults 0 0` 추가 후 `mount -a`
> **파티션 증설 (growpart):** `growpart /dev/vdb 1` -> `resize2fs /dev/vdb1`

### 3-2. LVM (Logical Volume Manager) 구성
1. PV/VG/LV 생성:
   `pvcreate /dev/vdd` -> `vgcreate datavg /dev/vdd` -> `lvcreate -l 100%FREE -n datalv datavg`
2. 파일시스템 생성 및 마운트:
   `mkfs.ext4 /dev/datavg/datalv` -> `mount /dev/datavg/datalv /data2`
3. 자동 마운트: `blkid`로 UUID 확인 후 `/etc/fstab`에 등록
> **LVM 증설:** `pvresize /dev/vdb` -> `lvextend -l +100%FREE /dev/datavg/datalv` -> `resize2fs /dev/datavg/datalv`

### 3-3. 스왑(Swap) 메모리 관리
* 스왑 확인: `free -h` 또는 `swapon -s`
* 스왑 생성/증설:
  `fallocate -l 32G /swapfile` -> `chmod 600 /swapfile` -> `mkswap /swapfile` -> `swapon /swapfile`
  (※ 자동 마운트를 위해 `/etc/fstab`에 `/swapfile swap swap defaults 0 0` 추가)

---

## 4. 파일 관리 및 시스템 설정
* 파일 위치 찾기: `find / -name [파일명]` 또는 `find / -type f -name [파일명] 2>/dev/null`
* 실행 파일 찾기: `which [실행파일명]`
* 파일 줄/단어 개수 세기: `wc -l [파일명]` (TCP 연결 수 확인: `netstat -ant | grep ESTABLISHED | wc -l`)
* 파일 비교 (`diff`): `diff -u 파일1 파일2` (-u: 직관적 맥락 포함, -i: 대소문자 무시, -w: 공백 무시)

## 5. 원격 파일 복사 및 실행 (SCP / SSH)
* 원격 서버에서 파일 가져오기: `scp -i [펨키] [user]@[ip]:/[경로] .`
* 원격 서버로 파일 보내기: `scp [전송할 파일] [전송 서버]:[경로]` (포트 지정 시 대문자 `-P`)
* 원격 서버 스크립트 실행: `ssh -t -i [펨키] [user]@[ip] "[실행 파일 위치]"`

## 6. DB 원격 접속 (MySQL / MariaDB)
* 접속: `mysql -h [IP] -P 3306 -u [계정] -p`
* 쿼리 바로 실행: `mysql -uroot -p -e "SELECT * FROM [테이블명];"`
* 백업 (Dump): `mysqldump -uroot -p scp_backup > scp_backup_dump.sql`

---

## 💡 트러블슈팅 사례 (실무 경험)
### 1. 망연계 서버 <-> 오브젝트 스토리지 통신 이슈
* **증상:** ping은 정상, TCP 443 연결 성공, 그러나 HTTPS 요청(curl) 실패 및 Packet Drop 발생.
* **원인:** MTU(Maximum Transmission Unit) 값 불일치. 서버(9000)와 망연계 구간(1500) 제한 충돌.
* **조치:** `sudo ip link set dev eth0 mtu 1500` 설정 후 통신 성공.# linux-study-note
