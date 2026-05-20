# 📝 TIL — 2026.05.17

## [Trouble Shooting] 서버 OOM으로 인한 반복 다운 문제

**문제**
`api.campushanjang.kr` 서버가 5/13부터 5/17까지 총 8번 다운됨
→ 원인은 OOM(Out of Memory), 리눅스 커널이 Spring Boot 프로세스를 강제 종료

**원인**
- t3.micro (RAM 1GB) 환경에서 Spring Boot JVM이 500~640MB 점유
- OS 기본 사용량까지 합치면 메모리 한계 초과
- `journalctl` 로그에서 `Failed with result 'oom-kill'` 반복 확인
- systemd가 자동 재시작하지만 근본 원인 미해결로 계속 반복

**조치**

1. **Swap 메모리 1GB 추가**
```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

2. **t3.micro → t3.small 업그레이드**
RAM 1GB → 2GB 확장 (월 약 $15 추가)

3. **CloudWatch CPU 알람 설정**
CPU 80% 초과 시 이메일 알림

4. **RateLimitConfig 메모리 누수 수정**
`ConcurrentHashMap` → `Caffeine Cache` 교체
→ 서버 업타임이 길어져도 메모리가 무한정 쌓이지 않도록 개선

**교훈**
> t3.micro에 Spring Boot 올릴 땐 Swap은 기본 세팅이다.
> 자동 재시작이 된다고 해결된 게 아니다. 로그부터 확인하자.


## 📌 추후 개선 예정

| 항목 | 내용 |
|------|------|
| JWT 파싱 최적화 | 요청마다 4번 파싱 → 1번으로 줄이기 |
| N+1 쿼리 개선 | 자정 스케줄러 DB 쿼리 폭발 방지 |
| RefreshToken 누적 문제 | 만료 토큰 정리 로직 추가 |
| EncryptionUtil 보안 취약점 | 고정 IV → 랜덤 IV 교체 (DB 마이그레이션 필요) |

---

# 📝 TIL — 2026.05.19

## [Trouble Shooting] 매칭 카드 중복 소진 문제

**문제**
서비스 오픈 전 가입 유저가 첫 매칭 진입 시 카드 10장이 이미 소진되어 있음

**원인**
프론트에서만 오픈 전 화면을 숨겼고, 백엔드 스케줄러는 오픈일과 무관하게 매일 자정 카드를 생성
→ 오픈 전 생성된 카드가 `recentIds`에 잡혀 후보 대부분 제외

**해결**
```
effectiveSince = max(가입일 - 30일, 서비스 오픈 전날)
```
DB 수정 없이 로직만으로 오픈 전 카드를 필터에서 제외, 30일 후 자동으로 정상 로직 전환

**교훈**
> 프론트에서 화면만 숨긴다고 끝이 아니다. 백엔드 스케줄러에도 오픈일 기준 가드가 필요하다.


## [Bug Fix] 중복 선택 시 횟수 이중 차감 문제

**문제**
- 같은 유저를 두 번 선택하면 횟수가 2번 차감됨
- 연락처 공개 후 X 버튼 누르면 다시 확인 불가

**원인**
`incrementSelectCount()`가 중복 선택 여부 체크 없이 무조건 실행

**해결**
```java
// SelectionRepository.java
existsBySelectorIdAndSelectedId() 추가

// MatchService.java
재선택 감지 → 카운트 차감 없이 동일 결과 반환
buildSelectResponse() 헬퍼 추출 → 연락처 재확인도 자연스럽게 해결
```
DB 구조 변경 없음, 무중단 배포 완료

**한계**
`score < 0.7` + 쪽지 미발송 상태에서 재선택 시 감지 불가
→ "쪽지 작성 중 이탈" 케이스에 가까워 현실적으로 수용

**교훈**
> 사용자 신고는 가장 빠른 QA다. 엣지 케이스는 실제 유저가 먼저 찾는다.

---
# 📝 TIL — 2026.05.20

## [Feature] 유저 나이 정보 표시 추가

**배경**
유저 피드백으로 매칭 카드에 나이 정보 표시 요청 수신

**구현**
- `birthYear` 필드를 카드 응답에 포함
- 프론트엔드에서 `~~년생` 형식으로 표시
