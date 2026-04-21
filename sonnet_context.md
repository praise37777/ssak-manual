# 싹싹컴퍼니 워드프레스 자동화 — 기술 컨텍스트 문서

> **⚠️ 이 문서는 Sonnet/Claude가 직원 질문에 정확히 답하기 위한 참조 문서입니다.**
> 새 채팅 시작 시 이 파일을 먼저 업로드한 후 직원 작업을 지원해주세요.

---

## 🎯 이 작업의 목적

싹싹컴퍼니는 여러 홈페이지(블루스카이, 블루워터, 시카고홈클리닝 등)를 운영하며, 각 홈페이지마다 동일한 자동화 시스템을 복제하여 연결해야 합니다. 이 문서는 **한 홈페이지(원본)를 기준으로 만들어진 자동화 시스템을 새 홈페이지에 복제·연결**하는 작업을 지원합니다.

### 주요 이해관계자
- **대표(형)**: 전체 시스템 설계, 민감한 보안 점검 담당
- **직원**: 매뉴얼 따라 실제 작업 수행. Sonnet에게 스크린샷으로 질문하며 진행
- **개발자**: n8n 복제 및 복잡한 문제 해결 지원 (미팅을 통해)

---

## 🏗️ 시스템 전체 구조

```
┌─────────────┐      ┌──────────────┐      ┌────────────────┐
│  n8n (5개)  │ ───▶ │  Make.com    │ ───▶ │  WordPress 사이트  │
│  워크플로   │ HTTP │  시나리오    │ REST │  (홈페이지)    │
└─────────────┘      └──────────────┘      └────────────────┘
      ▲                     ▲                       ▲
      │                     │                       │
      ▼                     ▼                       ▼
 ┌─────────────────────────────────────────────────────┐
 │             Google Sheets (키워드 DB)                │
 │             Google Drive (이미지/파일)               │
 └─────────────────────────────────────────────────────┘
```

### 역할 분담
- **n8n**: AI 글쓰기(Claude), 발행 여부 판단, 이미지 준비, Make에 데이터 전송
- **Make**: WordPress REST API 호출, 이미지 업로드, 카테고리/태그 생성, 글 발행
- **Google Sheets**: 키워드 대기열, 발행 기록, 오류 로그
- **Google Drive**: 참조 문서, 이미지 소스, 유튜브 임베드 시트

---

## 📁 n8n 워크플로 5개 — 완전 해부

### 원본 워크플로 ID (⚠️ 절대 수정 금지)
| 이름 | ID |
|------|------|
| 자동화 (원본) | `ma6GidVnMRG23oyv` |
| 발행 여부 설정 (원본) | `AYlTLMDKIrze6v6lWvVbg` |
| 클로드 파일 참조 (원본) | `frbGeRHCVcUoCm0sEHtAR` |
| 이미지 설정 (원본) | `kkTvO6G7kvxLw9g5AFX_L` |
| 실행 내역 삭제 (원본) | 독립 실행 (별도 복제 불필요) |

### copy 버전 ID (이미 복제 완료된 상태)
| 이름 | ID | 상태 |
|------|------|------|
| 자동화 copy | — (복제됨) | 서브워크플로 교체 완료 |
| 발행 여부 설정 copy | `Z5t3nLb54m9Ctde1` | — |
| 클로드 파일 참조 copy | `eEGtOV5rAvVCbBgN` | — |
| 이미지 설정 copy | `EGq9gJVFugxYkDob` | — |
| 실행 내역 삭제 copy | `8h5nwyj2pkh8J0Ds` | — |

### 1️⃣ 자동화 (메인 오케스트레이터)
- **노드 19개**
- **Schedule Trigger** → `발행 여부 설정` → `Switch1` → `클로드 파일 참조` → `Switch` → `이미지 설정` → `AI Agent` (Claude) → `Get HTML` → `Create a Post to Make.com` → 구글 드라이브 파일 삭제 6회
- **핵심 수정 대상**:
  - 서브워크플로 3개 (발행 여부 설정, 클로드 파일 참조, 이미지 설정) → copy 버전으로 교체
  - `Create a Post to Make.com` 노드 URL → 새 Make 웹훅으로 교체
  - `발행 중 설정` / `발행 중 설정1` 구글 시트 → 새 시트 ID로 교체

### 2️⃣ 발행 여부 설정
- **노드 20개**
- 구글 시트 7개 접근 (키워드 대기, 발행 개수, 발행 시간대, 유튜브 임베드 등)
- **구글 시트 탭 (gid)**:
  - `gid=0` — 메인 키워드 시트
  - `321749871` — 발행 개수
  - `1701077239` — 발행 시간대
  - `549469427` — 유튜브 임베드 영상

### 3️⃣ 클로드 파일 참조
- **노드 20개**
- 구글 드라이브에서 참조 파일 검색 → 다운로드 → Claude로 문서 분석
- **Anthropic Claude API 2회 호출** (Analyze document + Message a model)
- 구글 시트 탭: `149817661`, `321940459`, `gid=0`(오류 로그)
- Wait 노드 2개 (API rate limit 회피)

### 4️⃣ 이미지 설정
- **노드 33개** (가장 복잡)
- 구글 드라이브 폴더 6개 검색 → 이미지 선택 → 공유 권한 설정
- 이미지 폴더 비어있을 때 오류 처리 6개

### 5️⃣ 실행 내역 삭제
- **노드 5개**
- Schedule Trigger (3일마다) → 실행 이력 250개 조회 → 10개만 남기고 삭제
- **독립 실행** — 복제는 했지만 새 홈페이지 추가 시 수정 불필요

### 공통 사용 자원
- **구글 시트 문서 ID** (원본): `1r6QJlphYUIePNDI6p0sSDlz8yogL0r3uCj0LQc7qe00`
  - 새 사이트용으로 이 시트를 복사하고, 복사본 ID로 **모든 구글 시트 노드**를 교체해야 함

### Credential 의존성
- Google Sheets account — 모든 시트 노드 (다수)
- Google Drive account — 모든 드라이브 노드 (다수)
- Anthropic API — 클로드 파일 참조 (2개), 자동화 AI Agent (1개)
- n8n API — 실행 내역 삭제

---

## ⚙️ Make 시나리오 — 완전 해부

### 원본 시나리오 흐름 (총 11개 모듈)

```
[1] Webhooks (Custom webhook)
     ↓
[2] HTTP — Download file (구글 드라이브 이미지)
     ↓
[3] WordPress — Create a Media Item (이미지 업로드)
     ↓
[4] WordPress — Create a Category (카테고리 생성)
     ↓  (실패 시 onerror)
     → WordPress — Search Categories → Resume
     ↓
[5] Iterator (태그 배열 분리)
     ↓
[6] WordPress — Create a Tag (태그 생성)
     ↓  (실패 시 onerror)
     → WordPress — Search Tags → Resume
     ↓
[7] Array aggregator (태그 ID 모음)
     ↓
[8] WordPress — Create a Post (글 발행)
     ↓
[9] WordPress — Get a Post (발행된 글 조회)
     ↓
[10] Google Sheets — Search Rows
     ↓
[11] Google Sheets — Update a Row
```

### 원본 웹훅 URL (n8n이 바라보는 값)
```
https://hook.eu1.make.com/zvx1kqj9f4ftuu4mi49iix9lc56p8m4o
```

### 새 홈페이지 추가 시 교체해야 할 것
| 항목 | 개수 | 교체 방법 |
|------|------|------|
| WordPress Connection | 7곳 | 각 노드 클릭 → Connection 드롭다운 → 새 사이트 계정 선택 |
| Google Sheets Connection | 2곳 | 계정이 같다면 그대로, Sheet ID만 교체 |
| Google Sheets ID | 2곳 | Spreadsheet ID 항목에 새 시트 ID 붙여넣기 |
| Webhook URL | — | 시나리오 복제 시 자동으로 새 URL 생성됨. 이 URL을 n8n에 붙여넣기 |

### WordPress Connection 설정에 필요한 정보
- **Connection name**: 예) `블루스카이_연결`
- **WordPress REST API base url**: `https://www.새사이트주소.com/wp-json/`
- **API Key**: 워드프레스 → 설정 → Make Connector에서 복사

### ⚠️ Make Connector 플러그인 필수
- 새 WordPress 사이트에서 반드시 **Make Connector** 플러그인 설치 + 활성화
- 설치 후 [설정 > Make Connector] 메뉴에서 API Key 생성
- 이 Key가 없으면 Make가 워드프레스에 접근 불가

---

## 🔑 보안 점검 체크리스트

새 사이트 추가 전 반드시 확인:

### wp-config.php 악성코드 체크
- `@eval($_SERVER[...])` 같은 eval 사용 — **해킹 징후**
- 수상한 `base64_decode` 호출
- 알 수 없는 `include`, `require` 구문
- `HTTP_` 로 시작하는 커스텀 헤더 참조

### 기본 보안
- [ ] Wordfence 플러그인 설치 + 활성화 + 전체 스캔
- [ ] DB 비밀번호 변경 (wp-config.php에 평문으로 저장됨)
- [ ] 관리자 비밀번호 변경
- [ ] PHP 메모리 제한: `define('WP_MEMORY_LIMIT', '512M');`
- [ ] SSL 인증서 상태 확인

### PHP 설정 (HostGator cPanel)
- 파일 위치: `wp-config.php` (워드프레스 루트)
- 추가 위치: `/* Add any custom values between this line and the "stop editing" line. */` 바로 아래
- 추가 코드:
  ```php
  define('WP_MEMORY_LIMIT', '512M');
  ```

---

## 🚨 자주 발생하는 오류와 해결법

### 오류 1: "Workflow not found" (n8n)
- **원인**: 자동화 copy가 원본 워크플로를 호출 중 (서브워크플로 교체 안 됨)
- **해결**: 자동화 copy 열기 → `발행 여부 설정`/`클로드 파일 참조`/`이미지 설정` 노드 각각 클릭 → Workflow 드롭다운에서 copy 버전 선택

### 오류 2: "Credential is not set" (n8n ⚠️ 경고)
- **원인**: 워크플로 복제 후 Google 계정 연결이 풀림
- **해결**: 해당 노드 클릭 → 우측 패널 Credential 드롭다운 → 기존 계정 재선택 → 저장
- **영향 노드**: 모든 Google Sheets, Google Drive 노드 (다수)

### 오류 3: "Cannot read spreadsheet" (n8n)
- **원인**: 구글 시트 ID가 원본 시트를 바라보고 있음 OR 새 시트 권한 부족
- **해결 순서**:
  1. 새 시트 ID 확인 (메모장에 저장해둔 값)
  2. 해당 노드 Document ID 입력란에 ID 붙여넣기
  3. Sheet Name이 자동으로 로드되면 성공
  4. 안 되면 → 새 시트에 "편집자" 권한으로 Google 계정 공유 확인

### 오류 4: "WordPress connection failed" (Make)
- **원인 A**: Make Connector 플러그인 미설치/미활성화
- **원인 B**: API Key 오류 (복사할 때 공백 포함됨)
- **원인 C**: WordPress URL 끝에 `/` 누락 또는 `/wp-json/` 경로 누락
- **해결**:
  1. 워드프레스 관리자 → 플러그인 → Make Connector 활성 상태 확인
  2. 설정 → Make Connector → API Key 재복사
  3. URL은 `https://도메인.com/wp-json/` 형식 (끝에 슬래시 필수)

### 오류 5: "Invalid URL" / Make 시나리오 실행 안 됨
- **원인**: n8n의 `Create a Post to Make.com` 노드가 여전히 원본 웹훅 URL을 바라봄
- **해결**: 자동화 copy → `Create a Post to Make.com` 노드 → URL 칸 → 새 Make 시나리오 웹훅 URL로 교체

### 오류 6: Make 시나리오가 실행되는데 글이 안 올라감
- **원인 A**: WordPress Connection이 여전히 원본 사이트를 바라봄
- **원인 B**: 카테고리/태그 생성 실패 (권한 부족)
- **해결**:
  1. Make 시나리오에서 각 WordPress 모듈 클릭 → Connection 확인
  2. Executions 탭에서 실패한 모듈 클릭 → 에러 메시지 확인
  3. 워드프레스 사용자 권한이 "편집자" 이상인지 확인

### 오류 7: PHP 메모리 부족 (플러그인 활성화 실패)
- **증상**: Make Connector 플러그인 활성화 시 오류, 백엔드 화이트스크린
- **해결**: wp-config.php에 `define('WP_MEMORY_LIMIT', '512M');` 추가 후 재시도

---

## 🔄 Plan B / Plan C — 대안 방법

### Plan B: 개발자 말대로 폴더째 복사 방식
- n8n에서 Export → JSON 다운로드
- 새 워크스페이스 또는 새 폴더에 Import
- 서브워크플로 ID가 자동으로 재매핑되지 않으므로 **수동으로 copy 버전 연결 필요**

### Plan C: 파라미터화 방식 (장기적 추천)
- 현재: 사이트마다 n8n 5개 × Make 1개를 복제 → 관리 어려움
- 개선: n8n 워크플로를 그대로 두고, `siteId` 파라미터 하나로 분기
- Make에서 `siteId` 기반 라우팅
- **단점**: 기존 시스템 전면 리팩토링 필요. 신규 사이트 5개 미만이면 Plan A 권장.

### Plan D: 직접 만든 API 게이트웨이
- Cloud Run / Mac Mini에 웹훅 서버 구축
- 요청 받으면 `siteId`에 따라 적절한 WordPress로 라우팅
- n8n/Make 복제 불필요
- **단점**: 개발 공수 큼. 장기 사이트 대량 운영 시 유리.

---

## 📍 현재 진행 상황 (2026-04-22 기준)

### 완료
- ✅ n8n 테스트 폴더에 5개 copy 워크플로 생성
- ✅ 자동화 copy의 서브워크플로 3개 교체 완료
- ✅ 첫 번째 타겟 사이트: `www.blueskycleanair.com`
- ✅ HostGator cPanel에서 PHP 메모리 512M 추가 완료
- ✅ 해킹코드 (`@eval`) 발견 및 제거 완료
- ✅ Wordfence 설치 + 스캔 시작
- ✅ 안양 서브디렉토리 페이지 생성

### 진행 중
- 🟡 Wordfence 스캔 결과 대기
- 🟡 Make Connector 플러그인 활성화 (직원 확인 필요)
- 🟡 새 구글 시트 복사 및 ID 확보

### 보류 / 개발자 확인 대기
- ⏸ DB 비밀번호 변경 (파일에 평문 노출됨)
- ⏸ 관리자 비밀번호 변경
- ⏸ 다른 사이트 (ssakcompany, bluewater, chicago, awg) Wordfence 스캔

---

## 🧭 Sonnet에게 — 응답 가이드

직원이 이 매뉴얼을 보고 작업하다가 스크린샷과 함께 질문할 때:

1. **스크린샷 먼저 꼼꼼히 확인** — 어떤 화면의 어느 단계인지 파악
2. **이 문서의 "자주 발생하는 오류" 섹션 참조** — 비슷한 패턴 있는지 체크
3. **노드명/ID를 정확히 언급** — 이 문서의 실제 값 사용 (추측 금지)
4. **보안 경고에 민감하게 반응**:
   - `@eval`, `base64_decode`, 수상한 HTTP 헤더 참조 발견 시 → 즉시 대표에게 보고하라고 안내
   - 비밀번호, API Key, DB 정보가 포함된 파일 공유 요청 시 → 민감 정보 주의 안내
5. **확신 없으면 대표(형)에게 확인받도록 안내** — 특히 wp-config.php, Credential, DB 관련
6. **Plan B/C 제안** — Plan A가 안 먹힐 때 대안 제시

### 직원이 자주 하는 실수
- "활성화"를 "비활성화"로 잘못 누름 → 팝업에서 "취소" 눌러야 함
- 파일명이 한글이라 URL 404 → `index.html`로 리네임 유도
- 원본 URL을 지우지 않고 새 URL을 뒤에 덧붙임 → 전체 선택 후 교체 안내
- Make 시나리오 복제 후 이름 변경 안 함 → 사이트별 구분 어려움

### 직원이 물어보는 전형적인 질문 패턴
- "이거 맞나요?" + 스크린샷 → 화면의 상태 확인 후 피드백
- "다음 단계 뭐예요?" → 매뉴얼 단계 번호 안내
- "오류 떴어요" + 에러 스크린샷 → "자주 발생하는 오류" 참조해서 해결
- "활성화가 안 돼요" → 비활성화 팝업인지, PHP 메모리 문제인지 구분

---

## 📎 참고 자료 위치

- 매뉴얼 공개 URL: `https://praise37777.github.io/ssak-manual/`
- 대시보드 URL: `https://ssak-biz.web.app/` (내부용, 외부 공유 금지)
- Firebase Hosting: `~/ssak-biz-dashboard/` (Mac Mini)

---

*문서 버전: 1.0 · 최종 업데이트: 2026-04-22*
