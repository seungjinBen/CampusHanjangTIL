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

## [Bug Fix] 탈퇴 유저 카드 삭제 후 자동 보충 문제

**문제**
유저로부터 "같은 날 카드가 바뀌었다"는 제보 수신
→ 직접 계정으로 확인하니 1장만 새로운 카드, DB 조회 결과 탈퇴한 회원의 카드가 삭제되면서 보충된 것으로 확인

**원인**
- 탈퇴 시 `ON DELETE CASCADE`로 해당 유저의 카드가 DB에서 삭제됨
- `getTodayCards()`에서 `cards.size() < 10` 조건에 걸려 온디맨드로 1장 즉시 보충
- 결과적으로 당일 카드가 교체되는 것처럼 보이는 현상 발생

**해결**
`getTodayCards()`에서 온디맨드 보충 로직 제거
→ 카드는 자정 스케줄러만 생성, 탈퇴로 카드가 줄어도 당일은 그대로 유지

```java
// 제거된 로직
if (cards.size() < 10) {
    // 온디맨드 카드 보충 (제거)
}
```

**교훈**
> 유저 제보 → 직접 재현 → DB 확인 순서로 빠르게 원인을 특정할 수 있었다.  
> 편의를 위한 보충 로직이 오히려 UX를 해칠 수 있다.

---
# 📝 TIL — 2026.05.20

## [Feature] 유저 나이 정보 표시 추가

**배경**
유저 피드백으로 매칭 카드에 나이 정보 표시 요청 수신

**구현**
- `birthYear` 필드를 카드 응답에 포함
- 프론트엔드에서 `~~년생` 형식으로 표시


## [Bug Fix] 선택 횟수 비정상 차감 문제 (멱등성 누락 + 동시성)

**문제**
유저 2~3명으로부터 "한 번만 선택했는데 횟수가 3번 차감됐다"는 제보 수신
→ 로컬 테스트 6개 계정으로는 재현 불가 → AI 코드 분석으로 원인 특정

**원인**

버그 1 (주요) — 멱등성 가드 누락
- 이상형 일치율 < 75% 인 후보 선택 시 `NOTE_REQUIRED` 분기에서 `selectionRepository.save()` 미호출
- 재진입 가드 `existsBySelectorIdAndSelectedId()`가 항상 `false` 반환
- 더블탭 or 네트워크 재시도로 `POST /api/match/select` 중복 전송 시 `incrementSelectCount()`가 매번 실행됨

버그 2 — 경쟁 조건
- `getUser()`가 row lock 없이 조회 → 동시 요청 2개가 `dailySelectCount`를 동시에 읽으면 초과 선택 가능

버그 3 — sendNote() 무검증
- `select()` 호출 없이 `sendNote()` 직접 호출 가능
- 같은 상대에게 쪽지 중복 전송을 막지 않음

**해결**
```java
// 버그 1: NOTE_REQUIRED 분기에서도 Selection 즉시 저장
selectionRepository.save(new Selection(selectorId, selectedId, NOTE_SENT, matchScore));

// 버그 2: 비관적 락 적용
@Lock(LockModeType.PESSIMISTIC_WRITE)
findByIdForUpdate(Long id);

// 버그 3: sendNote() 진입 시 검증 추가
NOTE_SENT Selection 존재 확인 + 중복 쪽지 전송 차단
```

**피해 유저 추적 쿼리**
```sql
SELECT u.id, u.nickname, u.daily_select_count,
       COUNT(s.id) AS selection_count
FROM users u
LEFT JOIN selections s ON s.selector_id = u.id
    AND s.created_at >= CURRENT_DATE
WHERE u.daily_select_count > 0
GROUP BY u.id, u.nickname, u.daily_select_count
HAVING u.daily_select_count > COUNT(s.id)
ORDER BY (u.daily_select_count - COUNT(s.id)) DESC;
```
→ `daily_select_count > 실제 selections` 인 유저를 특정해 수동 보정 또는 자정 초기화 대기

**교훈**
> 로컬에서 재현 안 된다고 버그가 없는 게 아니다.  
> 더블탭·네트워크 재시도 등 실제 환경 변수를 항상 고려해야 한다.  
> 선택 횟수처럼 중요한 로직엔 멱등성과 동시성을 처음부터 설계에 포함시키자.
