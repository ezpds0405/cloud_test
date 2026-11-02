# 쓰레드(Threads) 육아템 체험 이벤트 탐색·응모 봇 설계안

> 상태: 설계 초안 (구현 전)
> 목표: Threads에 올라오는 육아용품 체험단/체험 이벤트 게시글을 자동으로 찾아내고, 응모 과정을 최대한 자동화한다.

---

## 1. 먼저 짚어야 할 제약 (설계 방향을 결정하는 요소)

| 항목 | 내용 | 설계에 주는 영향 |
|---|---|---|
| Threads 이용약관 | Meta는 허가 없는 자동 수집(스크래핑)과 자동화된 계정 활동(봇 팔로우·좋아요·댓글 도배)을 금지. 위반 시 계정 제한/정지 | **비공식 웹 크롤링(Selenium/Playwright로 로그인 후 긁기) 대신 공식 Threads API 사용** |
| 이벤트 주최측 규칙 | 체험단 이벤트 상당수가 "복붙 댓글·매크로 응모는 당첨 제외"를 명시. 성의 있는 신청 사유가 당첨에 유리 | **완전 자동 응모 X → "자동 탐색 + 초안 생성 + 사람 1클릭 승인"** |
| 공식 API 기능 범위 | 키워드 검색, 게시글/답글 조회, 답글(댓글) 작성, 멘션 조회 가능. **팔로우·좋아요 API는 없음** | 댓글 응모는 API로 자동화, 팔로우/좋아요/외부 신청폼은 딥링크로 사용자에게 넘김 |
| 개인정보 | 응모 시 이름·연락처·주소·아이 월령 등을 요구하는 경우가 많음 | 개인정보는 로컬 암호화 저장, 외부 폼 자동 제출은 하지 않음(사기성 이벤트 위험) |

**결론:** "크롤링 + 무인 자동응모"를 그대로 만들면 계정 정지와 당첨 제외 위험이 커서 실익이 없다.
**"Threads 공식 API로 자동 탐색 → AI가 이벤트 판별·조건 추출·응모 댓글 초안 작성 → 메신저로 알림 → 사용자가 버튼 한 번 누르면 API로 응모 댓글 게시"** 구조를 권장한다. 사람이 하는 일은 확인 1클릭(+필요 시 팔로우)으로 줄어든다.

---

## 2. 전체 구조

```
 ┌──────────────┐   주기 실행(30~60분)
 │  Scheduler   │──────────────────────────────┐
 └──────────────┘                              ▼
 ┌──────────────────────┐   ┌───────────────────────────┐
 │ 1. Collector         │──▶│ 2. Classifier / Extractor │
 │ Threads keyword_search│   │ 규칙 필터 → LLM 구조화 추출 │
 └──────────────────────┘   └─────────────┬─────────────┘
                                          ▼
 ┌──────────────────────┐   ┌───────────────────────────┐
 │ 4. Notifier          │◀──│ 3. Scorer / Filter        │
 │ 텔레그램 등 + 버튼     │   │ 월령·마감·중복·사기 필터    │
 └─────────┬────────────┘   └───────────────────────────┘
           │ [응모] 클릭
           ▼
 ┌──────────────────────┐   ┌───────────────────────────┐
 │ 5. Entry Executor    │──▶│ 6. Tracker                │
 │ 답글 API로 댓글 게시   │   │ 마감·발표·리뷰기한 리마인드 │
 └──────────────────────┘   └───────────────────────────┘
            모든 단계 상태는 SQLite에 저장
```

---

## 3. 단계별 상세

### 3.1 Collector — 게시글 수집

- **API:** Threads Graph API 키워드 검색 (`GET https://graph.threads.net/v1.0/keyword_search`)
  - 파라미터: `q`(키워드), `search_type=RECENT`, `since`/`until`, `fields=id,text,permalink,timestamp,username,media_type,media_url`
  - 권한: `threads_basic`, `threads_keyword_search` (공개 게시글 전체를 검색하려면 **Meta 앱 검수(App Review)** 필요. 검수 전에는 본인 게시글만 검색되는 제한이 있음)
  - 검색 호출에는 사용자별 쿼터(롤링 기간당 횟수 제한)가 있으므로 키워드 수 × 실행 주기를 쿼터 안에 맞춘다. 정확한 수치는 착수 시 공식 문서로 확인.
- **키워드 세트 (설정 파일로 관리):**
  - 이벤트성: `체험단`, `체험 이벤트`, `체험단 모집`, `리뷰어 모집`, `서포터즈 모집`, `무료체험`, `댓글 이벤트`, `기프티`
  - 육아 도메인: `육아템`, `아기`, `신생아`, `유아`, `기저귀`, `젖병`, `이유식`, `유모차`, `카시트`, `아기띠`, `물티슈`, `분유`, `턱받이`
  - 조합 쿼리: `육아 체험단`, `아기 체험단`, `기저귀 체험`, `이유식 체험단` 등 (조합으로 검색하면 쿼터 대비 정확도가 높음)
- **보조 수집 경로:** 사용자가 Threads 앱에서 게시글 링크를 봇에 "공유"하면 같은 파이프라인으로 처리 (검색에 안 잡히는 게시글 보완).
- 수집 결과는 `posts` 테이블에 `post_id` 기준 upsert로 저장해 중복 처리를 막는다.

### 3.2 Classifier / Extractor — 이벤트 판별 및 조건 추출

1. **1차 규칙 필터 (무료·빠름):** 본문에 이벤트 키워드 + 육아 키워드가 모두 있는지, "당첨/모집/응모/마감" 같은 단어가 있는지 확인. 통과하지 못하면 버림.
2. **2차 LLM 구조화 추출:** 통과한 글만 Claude API(분류는 저가 모델 `claude-haiku-5-5`로 충분)에 넘겨 JSON으로 뽑는다.

```json
{
  "is_trial_event": true,
  "brand": "OO베이비",
  "product": "신생아 속싸개",
  "category": "의류/침구",
  "target_child_age": {"min_months": 0, "max_months": 6},
  "winners": 10,
  "deadline": "2026-11-10T23:59:00+09:00",
  "announce_date": "2026-11-12",
  "requirements": {
    "follow_accounts": ["@oobaby_official"],
    "like": true,
    "comment": true,
    "comment_guide": "아이 월령과 신청 이유를 적어주세요",
    "tag_friends": 2,
    "repost": false,
    "external_form_url": null,
    "review_obligation": "수령 후 7일 내 쓰레드/인스타 후기"
  },
  "red_flags": [],
  "confidence": 0.92
}
```

- 게시글이 이미지 위주(공지 카드 이미지)인 경우 `media_url` 이미지를 함께 넘겨 비전으로 조건을 읽는다.
- `confidence`가 낮으면 "확인 필요"로 분류해 알림만 보낸다.

### 3.3 Scorer / Filter — 응모 대상 선정

- **자격 필터:** 사용자 프로필(아이 출생월, 거주 지역, 이미 가진 제품)과 `target_child_age`·지역 조건 비교
- **시간 필터:** 마감 지난 글 제외, 마감 임박 순 정렬
- **중복 필터:** 같은 브랜드·같은 제품 재게시, 이미 응모한 이벤트 제외
- **사기/위험 필터 (`red_flags`):** 배송비·선입금 요구, DM으로 주민번호/계좌 요구, 생성된 지 얼마 안 된 계정, 외부 단축URL만 있는 경우 → 응모 대상에서 제외하고 경고만 표시
- **점수:** 제품 관심도(사용자 선호 카테고리) × 당첨 인원 × 응모 난이도(요구 조건 수 역수) 로 우선순위 계산

### 3.4 Notifier — 알림 및 승인

- 채널: 텔레그램 봇 권장(인라인 버튼·무료·API 간단). 카카오톡은 비즈 채널이 필요해 개인용으로는 번거로움.
- 메시지 예시:

```
🍼 [마감 D-3] OO베이비 신생아 속싸개 체험단 (10명)
조건: @oobaby_official 팔로우 · 좋아요 · 댓글(월령+사유) · 친구 2명 태그
대상: 0~6개월 ✅ (우리 아이 4개월)

📝 댓글 초안:
"4개월 아기 키우고 있어요! 요즘 모로반사 때문에 자꾸 깨서 속싸개를 찾던 중이었는데,
OO베이비 제품은 원단이 얇고 신축성이 좋다고 들어 꼭 써보고 싶어요 🙏 @친구1 @친구2"

[✅ 이대로 응모] [✏️ 수정] [⏭ 건너뛰기] [🔗 원문 열기]
```

- **댓글 초안 생성:** 사용자 프로필 + 제품 정보 + 주최측 가이드로 매번 다른 문장을 생성(복붙 방지). 태그할 친구 목록은 사용자가 미리 등록해 두고 순환 사용.

### 3.5 Entry Executor — 응모 실행

| 조건 | 처리 방식 |
|---|---|
| 댓글 | 공식 API로 답글 게시: `POST /me/threads` (`media_type=TEXT`, `text`, `reply_to_id=<post_id>`) → `POST /me/threads_publish` (`creation_id`). 권한 `threads_content_publish`, `threads_manage_replies` |
| 리포스트/인용 | 공식 API 지원 범위에서 처리, 미지원 시 딥링크 |
| 팔로우 / 좋아요 | API 미지원 → 알림 메시지의 "원문 열기" 딥링크로 사용자가 직접 탭(몇 초면 끝남). 체크박스로 완료 표시 |
| 외부 신청폼(구글폼·네이버폼) | 링크 제공 + 저장된 프로필 정보를 복사하기 쉽게 보여주기만 함. 자동 제출 안 함 |

- 실행 간격에 무작위 지연을 두고, 하루 응모 댓글 수 상한(예: 10~15건)을 설정해 스팸 판정을 피한다.
- 실행 결과(성공/실패, 게시된 답글 ID)를 `applications` 테이블에 기록.

### 3.6 Tracker — 사후 관리

- 마감 전 미완료 조건(팔로우 체크 안 됨 등) 리마인드
- 발표일에 주최 게시글의 답글/당첨 공지 글을 다시 조회하고, 멘션 API(`threads_manage_mentions`)로 내 계정이 언급됐는지 확인 → 당첨 알림
- 당첨 시 리뷰 의무 기한 리마인드 (체험단은 후기 미작성 시 다음 이벤트 블랙리스트가 되는 경우가 많음)

---

## 4. 데이터 모델 (SQLite)

```sql
CREATE TABLE posts (
  post_id TEXT PRIMARY KEY, username TEXT, text TEXT, permalink TEXT,
  posted_at TEXT, fetched_at TEXT, matched_keyword TEXT
);
CREATE TABLE events (
  post_id TEXT PRIMARY KEY REFERENCES posts(post_id),
  brand TEXT, product TEXT, category TEXT,
  min_months INT, max_months INT, winners INT,
  deadline TEXT, announce_date TEXT,
  requirements_json TEXT, red_flags_json TEXT,
  confidence REAL, score REAL,
  status TEXT  -- new / notified / skipped / applied / won / lost / expired
);
CREATE TABLE applications (
  post_id TEXT PRIMARY KEY REFERENCES events(post_id),
  comment_text TEXT, reply_id TEXT,
  followed INT DEFAULT 0, liked INT DEFAULT 0, form_done INT DEFAULT 0,
  applied_at TEXT, result TEXT, review_due TEXT
);
CREATE TABLE profile (key TEXT PRIMARY KEY, value TEXT); -- 아이 출생월, 지역, 선호 카테고리, 태그 친구 목록 등
```

---

## 5. 기술 스택 제안

- 언어: Python 3.12
- HTTP: `httpx` (Threads Graph API 호출)
- 스케줄링: `APScheduler` (상시 구동 시) 또는 GitHub Actions cron / 소형 VPS cron
- 저장소: SQLite (+ 개인정보 컬럼은 `cryptography` Fernet으로 암호화)
- LLM: Anthropic Python SDK — 분류·추출 `claude-haiku-5-5`, 댓글 초안 `claude-sonnet-5-5`
- 알림: `python-telegram-bot` (인라인 버튼 콜백으로 승인 처리 → 콜백 수신 위해 상시 구동 또는 webhook 필요)
- 비밀값: Threads 장기 액세스 토큰(60일, 만료 전 갱신 잡 필요), Anthropic API 키, 텔레그램 봇 토큰 → 환경변수/시크릿

### 디렉터리 구조(안)

```
bot/
  config.yaml          # 키워드, 하루 응모 상한, 실행 주기
  collector.py         # keyword_search 호출, posts upsert
  extractor.py         # 규칙 필터 + LLM 구조화 추출
  scorer.py            # 자격/중복/사기 필터, 점수
  notifier.py          # 텔레그램 메시지·버튼·콜백
  executor.py          # 답글 게시
  tracker.py           # 리마인드, 당첨 확인
  threads_client.py    # 토큰 갱신, API 래퍼, 쿼터 관리
  db.py
tests/
```

---

## 6. 사전 준비 (Meta 쪽)

1. Meta 개발자 계정 생성 → 앱 생성 → "Threads API" 유즈케이스 추가
2. 본인 Threads 계정을 테스터로 등록 후 OAuth로 액세스 토큰 발급, 장기 토큰으로 교환
3. 권한: `threads_basic`, `threads_keyword_search`, `threads_content_publish`, `threads_manage_replies`, `threads_read_replies`, `threads_manage_mentions`
4. 공개 게시글 검색을 위해 `threads_keyword_search` 앱 검수 신청 (용도 설명 + 화면 녹화 필요). **검수 통과 여부가 프로젝트 성패를 좌우하므로 가장 먼저 진행**

---

## 7. 단계별 구현 계획

| 단계 | 범위 | 완료 기준 |
|---|---|---|
| **Phase 0** | Meta 앱 생성, 토큰 발급, 검수 신청 | 내 토큰으로 keyword_search 호출 성공 |
| **Phase 1 (MVP)** | Collector + 규칙 필터 + LLM 추출 + 텔레그램 알림 | 하루 1회 이상 신규 육아 체험 이벤트가 정리되어 알림으로 옴 |
| **Phase 2** | 댓글 초안 생성 + 승인 버튼 + 답글 API 게시 | 버튼 한 번으로 응모 댓글이 실제 게시됨 |
| **Phase 3** | Tracker(리마인드·당첨 확인·리뷰 기한) | 발표일에 당첨 여부 알림 |
| **Phase 4** | 점수 개선(당첨 이력 기반 학습), 웹 대시보드 | 선택 사항 |

---

## 8. 검토했지만 채택하지 않은 대안

| 대안 | 기각 이유 |
|---|---|
| Playwright/Selenium으로 로그인 후 웹 크롤링 + 자동 클릭 | 약관 위반, 봇 탐지 시 계정 정지, 프론트 변경 시 수시로 깨짐 |
| 비공식 사설 API 라이브러리(내부 GraphQL 호출) | 위와 동일 + 인증정보 유출 위험 |
| 승인 없는 완전 무인 자동응모 | 복붙 댓글로 당첨 제외, 스팸 판정, 사기성 이벤트에 개인정보 노출 위험 |
| 다계정으로 응모 | 대부분 이벤트에서 부정 응모로 간주, 약관 위반 |

---

## 9. 열린 질문

- 공개 검색 앱 검수가 반려될 경우 대안: 브랜드 계정 목록을 미리 등록해 두고 해당 계정의 게시물만 모니터링하는 방식(계정 기반 조회) 가능 여부 확인 필요
- 인스타그램·네이버 블로그/카페 체험단까지 확장할지 (Threads 글이 해당 채널로 링크하는 경우가 많음)
- 알림 채널: 텔레그램 vs 카카오톡 vs 이메일
