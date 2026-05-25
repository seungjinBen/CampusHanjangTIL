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

## [Bug Fix] 선택 횟수 비정상 차감 문제 (멱등성 누락 + 동시성)

**문제**
유저 2~3명으로부터 "한 번만 선택했는데 횟수가 3번 차감됐다"는 제보 수신
→ 로컬 테스트 6개 계정으로는 재현 불가 → `NOTE_REQUIRED` 분기 코드 분석으로 멱등성 가드 누락 및 row lock 미적용 특정

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


## [장애 대응] 선택 횟수 과차감 피해 유저 DB 보정

**상황**
버그 수정 배포 후, 이미 피해를 입은 유저들의 `daily_select_count` 보정 필요

**피해 유저 특정 쿼리**
```sql
WITH actual_today AS (
    SELECT selector_id, COUNT(*) AS cnt
    FROM selections
    WHERE created_at >= CURRENT_DATE
    GROUP BY selector_id
)
SELECT u.id, u.nickname,
       u.daily_select_count AS current_count,
       COALESCE(a.cnt, 0) AS actual_count,
       u.daily_select_count - COALESCE(a.cnt, 0) AS over_charged
FROM users u
LEFT JOIN actual_today a ON a.selector_id = u.id
WHERE u.daily_select_count > COALESCE(a.cnt, 0)
ORDER BY over_charged DESC;
```

**보정 쿼리**
```sql
WITH actual_today AS (
    SELECT selector_id, COUNT(*) AS cnt
    FROM selections
    WHERE created_at >= CURRENT_DATE
    GROUP BY selector_id
)
UPDATE users
SET daily_select_count = COALESCE(
    (SELECT cnt FROM actual_today WHERE selector_id = users.id), 0
)
WHERE daily_select_count > COALESCE(
    (SELECT cnt FROM actual_today WHERE selector_id = users.id), 0
)
RETURNING id, nickname, daily_select_count AS corrected_count;
```

**설계 포인트**
- `CURRENT_DATE` → 오늘 자정 기준으로만 집계
- `COALESCE(..., 0)` → 오늘 선택이 아예 없는 유저도 처리
- `WHERE daily_select_count > actual` → 정상 유저는 건드리지 않음
- `RETURNING` → 보정된 유저 목록 즉시 확인 가능

**교훈**
> 버그 수정만으로 끝이 아니다.  
> 이미 피해를 입은 유저까지 복구해야 진짜 대응이 완료된다.  
> 자정 초기화 스케줄러 덕분에 무중단으로 보정할 수 있었다.


## [Performance] Firebase Storage 이미지 로딩 속도 개선

**문제**
매칭 카드 이미지 10장이 매번 느리게 로딩됨

**원인**
Firebase Storage 기본 Cache-Control이 `private, max-age=0`으로 설정되어 있어 매 요청마다 Firebase 서버에서 새로 받아오고 있었음

**해결**
```java
// 업로드 시 Cache-Control 명시적 설정
BlobInfo.newBuilder(blobId)
    .setContentType("image/jpeg")
    .setCacheControl("public, max-age=2592000") // 30일
    .build();
```
신규 업로드부터 브라우저 캐시 적용, 두 번째 방문부터 즉시 로드

**교훈**
> Firebase Storage 기본값은 캐시를 하지 않는다.  
> 토이프로젝트에선 혼자 쓰니 몰랐지만, 실서비스에서 유저가 늘어나자 바로 체감됐다.  
> 실서비스 운영이 토이프로젝트에선 발견 못한 문제를 드러낸다.

---
📝 TIL — 2026.05.25
## [Performance] N+1 쿼리 개선 — k6 부하테스트 검증

**문제**
매칭 카드 조회 시 유저 1명당 22개 쿼리 발생
500VU 환경에서 p95 응답시간 23,527ms, 에러율 0.83%

**원인**
```
getTodayCards: 1(user) + 1(cards) + 10(traits×10) + 10(photos×10) = 22쿼리
getReceivedNotes: 1(notes) + N(traits) + N(photos) = 2N+1쿼리
```

**해결**
```java
// Before: 카드 N장 → N번 개별 조회
buildCardDto(DailyCard) × N

// After: 배치 IN 쿼리로 한 번에 조회
photoRepository.findByUserIdIn(ids)       // 1쿼리
userTraitRepository.findByUserIdIn(ids)   // 1쿼리
Map으로 변환 후 루프에서 O(1) 조회
```

**결과 (500VU 기준)**

| 지표 | 개선 전 | 개선 후 | 개선율 |
|------|---------|---------|--------|
| p95 응답시간 | 23,527ms | 8,873ms | **62.3% 단축** |
| RPS | 62.3 | 128.0 | **105% 향상** |
| 에러율 | 0.83% | 0.00% | **완전 해소** |
| 총 처리량 | 10,670건 | 21,431건 | **2배 증가** |

**교훈**
> 쿼리 수 82% 감소가 커넥션 풀 경합을 해소해  
> 전 엔드포인트에 걸쳐 복합 성능 향상 효과를 만들었다.
