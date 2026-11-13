# 검색·정렬·페이지네이션 게시판 개선하기

웹서비스설계 9주차 실습 과제입니다. 첨부 예제를 실행하여 **기능 검증 → 코드 개선 → 재검증 및 결과 설명** 순서로 진행했습니다.

- 실습 대상: 저장소 최상위의 `server.js`, `index.html`입니다.
- 데이터: 서버 실행 시 생성하는 인메모리 게시글 1,000건을 사용합니다.
- 자료: [과제 요구사항](%EC%9A%94%EA%B5%AC%EC%82%AC%ED%95%AD.png), [제출 체크리스트](%EC%A0%9C%EC%B6%9C%20%EC%B2%B4%ED%81%AC%EB%A6%AC%EC%8A%A4%ED%8A%B8.png)

## 목차

1. [실행 방법](#run)
2. [구현 기능과 API](#api)
3. [기존 코드 대비 수정 내용](#changes)
4. [항목별 검증 결과](#verification)
5. [핵심 개념](#concepts)
6. [AI 활용과 직접 검증](#ai)
7. [전체 증거 사진 74장](#evidence)
8. [제출 확인](#submission)

<a id="run"></a>

## 🚀 빠른 실행 방법

Node.js 18 이상과 npm이 필요합니다. 저장소를 복제한 다음 `server.js`가 있는 저장소 최상위 폴더에서 실행합니다.

```bash
# 1. 저장소 클론
git clone https://github.com/kwonjunsun9/26WSD-practice-02.git

# 2. 저장소 이동
cd 26WSD-practice-02

# 3. 의존성 설치
npm install express

# 4. 백엔드 서버 구동
node server.js
``` 

 이 후 브라우저에서 **http://localhost:3000**에 접속합니다. 서버를 종료할 때는 실행한 터미널에서 `Ctrl+C`를 누릅니다. 의존성 설치 시 생성되는 `package.json`과 `package-lock.json`은 함께 관리하고, `node_modules`는 제출 대상에서 제외합니다.

```text
26WSD-practice-02/
├── README.md
├── server.js                 # 데이터 생성·검색·정렬·페이지네이션 API
├── index.html                # 게시판 화면·요청 취소·URL 상태 복원
├── package.json              # 의존성 설치 시 생성합니다.
├── package-lock.json         # 의존성 설치 시 생성합니다.
├── 웹서비스설계 실습2 보고서 - 202219348 권준성.pdf
├── 요구사항.png
├── 제출 체크리스트.png
└── evidence/
    ├── 기능검증/              # 기존 동작과 각 기능의 개선 전 상태
    └── 코드개선/              # 변경 코드(code)·실행 결과(photo)
```

조회수는 서버를 다시 시작할 때 무작위로 생성됩니다. 전후 비교에서는 같은 서버 실행 중의 조회수와 정렬 규칙을 확인했습니다. 


<a id="api"></a>

## 2. 구현 기능과 API

### 2.1 오프셋 API

```http
GET /api/posts?page=1&limit=10&sort=latest&search=Redis
```

| 파라미터 | 동작 |
|---|---|
| `page` | 기본값은 1입니다. `0`, 음수, 숫자가 아닌 값은 1로 보정하고, 마지막 페이지보다 크면 마지막 페이지로 제한합니다. |
| `limit` | 기본값은 10입니다. 양수 입력의 상한은 100이며 `limit=1000`은 100으로 보정합니다. |
| `sort` | `latest`, `oldest`, `views`만 허용합니다. 잘못된 값과 `__proto__`, `constructor`는 `latest`로 처리합니다. |
| `search` | 제목 또는 본문에 검색어가 포함된 글을 반환합니다. 공백을 정리하고 대소문자를 구분하지 않습니다. |

응답은 `items`와 `meta`로 구성됩니다. `meta`에는 `page`, `limit`, `sort`, `totalCount`, `totalPages`, `hasNext`, `hasPrev`, `executionTimeMs`가 들어 있습니다. 결과가 없으면 `items=[]`, `totalCount=0`, `totalPages=1`로 반환합니다.

| 정렬 키 | 실제 정렬 기준 |
|---|---|
| `latest` | 작성일 내림차순 → 같은 작성일이면 ID 내림차순입니다. |
| `oldest` | 작성일 오름차순 → 같은 작성일이면 ID 오름차순입니다. |
| `views` | 조회수 내림차순 → 같은 조회수이면 ID 내림차순입니다. |

### 2.2 커서 API

```http
GET /api/posts/cursor?limit=10&search=Redis
GET /api/posts/cursor?limit=10&search=Redis&cursor=883
```

커서는 **ID 내림차순으로 고정**합니다. 검색 조건과 `id < cursor` 조건을 적용한 다음 `limit + 1`개를 확인하여 `hasNext`를 판단합니다. 첫 요청에는 `cursor`를 생략하고, 다음 요청에는 응답의 `nextCursor`를 사용합니다. 커서 API의 `limit` 기본값은 10이고 상한은 50입니다.

응답의 `meta`에는 `cursor`, `nextCursor`, `limit`, `hasNext`, `executionTimeMs`가 들어 있습니다. `nextCursor`는 실제 반환한 마지막 글의 ID입니다. 반환할 글이 없으면 `nextCursor=null`입니다. 마지막 묶음에 글이 남아 있으면 `hasNext=false`여도 `nextCursor`에 마지막 ID가 들어갈 수 있으므로 종료 여부는 **`hasNext`로 판단**합니다.

### 2.3 화면 동작

오프셋 모드는 페이지 번호를 사용하고, 커서 모드는 더 보기 버튼으로 목록을 누적합니다. 검색·정렬 변경 시 오프셋은 1페이지로 이동합니다. 커서 검색어 변경 시 누적 목록과 커서를 초기화하며, 커서 모드에서는 정렬 선택을 비활성화하고 ID 내림차순 고정을 표시합니다. `hasNext=false`이면 더 보기 버튼을 비활성화합니다.

검색 입력에는 300ms 디바운스와 `AbortController`를 적용합니다. 오프셋 모드의 검색어·정렬·페이지·limit은 `URLSearchParams`로 동기화하고 새로고침과 `popstate`에서 복원합니다.

<a id="changes"></a>

## 3. 기존 코드 대비 수정 내용

원래 정상인 기능과 직접 수정한 기능을 구분했습니다. **오프셋 기본 계산·입력값 보정, 제목·본문 검색과 빈 결과 처리, 조회수 동점 정렬, 300ms 디바운스, AbortError 처리**는 기존 동작을 유지하며 검증했습니다.

| 위치 | 개선 전 | 직접 반영한 내용 | 변경 코드 |
|---|---|---|---|
| `server.js` 정렬 | 최신순·오래된순이 ID만 비교했습니다. | 작성일을 먼저 비교하고 같은 날짜에서는 ID를 비교합니다. | [정렬 비교 함수](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%B5%9C%EC%8B%A0%EC%88%9C%2C%EC%98%A4%EB%9E%98%EB%90%9C%EC%88%9C%2C%EC%A1%B0%ED%9A%8C%EC%88%98%EC%88%9C%20%EA%B2%80%EC%A6%9D.png) |
| `server.js` 정렬 키 | 객체의 상속 속성까지 조회할 수 있었습니다. | 문자열 여부와 `Object.hasOwn`으로 허용 키만 선택합니다. | [화이트리스트 검사](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%A0%95%EB%A0%AC%ED%82%A4%20%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-1.png) |
| `server.js` 응답 | 적용된 정렬 키가 meta에 없었습니다. | `sort: sortKey`를 추가합니다. 실제 차단은 화이트리스트 검사에서 수행합니다. | [meta.sort 추가](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%A0%95%EB%A0%AC%ED%82%A4%20%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-2.png) |
| `server.js` 커서 검색 | 제목·본문 검색 조건이 없었습니다. | 검색·커서 조건을 함께 적용한 결과에 기존 `limit + 1` 처리를 적용합니다. | [검색 필터 추가](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9CAPI%20%EC%A0%9C%EB%AA%A9%2C%EB%B3%B8%EB%AC%B8%20%EC%B6%94%EA%B0%80.png) |
| `index.html` 커서 요청 | 검색창의 값이 커서 요청에 전달되지 않았습니다. | 요청의 `URLSearchParams`에 `search`를 추가합니다. | [검색어 전달](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9CAPI%20%EA%B2%80%EC%83%89%EC%B6%94%EA%B0%80.png) |
| `index.html` 커서 정렬 | 화면에서 정렬을 선택할 수 있어 실제 기준과 혼동할 수 있었습니다. | 고정 옵션을 표시하고 커서 모드에서 선택을 막습니다. | [모드별 선택 제어](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20ID%EB%82%B4%EB%A6%BC%EC%B0%A8%EC%88%9C%EA%B3%A0%EC%A0%95.png) · [고정 옵션](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EA%B3%A0%EC%A0%95%EA%B8%B0%EC%A4%80%20%ED%91%9C%EC%8B%9C.png) |
| `index.html` 요청 취소 | 새 fetch를 시작할 때 이전 요청을 취소했습니다. | 입력 즉시 취소하고 JSON 파싱 뒤 `signal.aborted`를 검사합니다. | [입력 즉시 취소](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/code/%EC%9D%B4%EC%A0%84%EC%9A%94%EC%B2%AD%EC%B7%A8%EC%86%8C.png) · [응답 반영 차단](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/code/%EC%B7%A8%EC%86%8C%EB%90%9C%EC%9A%94%EC%B2%AD%EC%9D%98%20%EC%9D%91%EB%8B%B5%20%EB%B0%98%EC%98%81%EC%95%88%ED%95%A8.png) |
| `index.html` 입력 처리 | 입력 직후 정렬 변경 시 검색어가 유실되거나 수동 조회와 예약 요청이 중복될 수 있었습니다. | 공통 `applySearch()`에서 타이머를 정리하고 현재 입력을 읽습니다. | [정렬 이벤트](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EA%B2%80%EC%83%89%EC%96%B4%20%EC%9C%A0%EC%A7%80%ED%95%98%EB%A9%B0%20%EC%A1%B0%ED%9A%8C.png) · [공통 조회 함수](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%A1%B0%ED%9A%8C%20%EC%8B%9C%20%EC%A4%91%EB%B3%B5%EC%9A%94%EC%B2%AD%20%EB%B0%A9%EC%A7%80.png) |
| `index.html` URL 복원 | 서버 보정값과 주소·선택창이 어긋날 수 있었습니다. | 응답 meta를 상태와 URL에 반영하고 최초 로드·뒤로 가기를 공통 함수로 복원합니다. | [보정값 동기화](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/code/%EC%9E%98%EB%AA%BB%EB%90%9C%20URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90.png) · [공통 복원 이벤트](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/code/%EC%83%88%EB%A1%9C%EA%B3%A0%EC%B9%A8%2C%20%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%ED%9B%84%20%EA%B2%B0%EA%B3%BC%EC%9D%BC%EC%B9%98.png) |


<a id="verification"></a>

## 4. 항목별 검증 결과

사진은 해당 기능을 보완하는 과정에서 순서대로 촬영했습니다. `기능검증` 폴더에도 앞선 분야의 개선이 반영된 뒤 촬영한 사진이 있으며, `코드개선` 폴더에도 정렬 UI 고정 전의 중간 단계 사진이 있습니다. 최종 동작은 아래 결과와 연결된 자료를 기준으로 확인합니다. 전체 사진의 용도는 [74장 목록](#evidence)에 따로 정리했습니다.

### 4.1 검색·정렬 — 25점

| 검증 입력·행동 | 기대 결과 | 실제 결과·비교 근거 |
|---|---|---|
| `Redis`, 최신순, 1페이지 | Redis 제목이 포함된 글을 표시합니다. | 기존 검색을 유지하고 작성일 정렬을 적용했습니다. 첫 ID는 원본 991에서 개선 후 919로 바뀌었습니다. [전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%A0%9C%EB%AA%A9%EA%B2%80%EC%83%89.png) · [후](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%A0%9C%EB%AA%A9%EA%B2%80%EC%83%89.png) |
| `상세 본문` 검색 | 제목에 없어도 본문이 일치하면 반환합니다. | JSON content에 검색어가 있고 1,000건이 검색됩니다. [본문 내용](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EB%B3%B8%EB%AC%B8%20%EA%B2%80%EC%83%89%20%EA%B2%B0%EA%B3%BC.png) · [meta](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EB%B3%B8%EB%AC%B8%20%EA%B2%80%EC%83%89.png) |
| 없는 검색어 | 빈 목록 안내와 `items=[]`를 표시합니다. | 안내 문구, `totalCount=0`, `totalPages=1`을 확인했습니다. [기존 동작](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%97%86%EB%8A%94%EA%B2%80%EC%83%89%EC%96%B4%EC%9E%85%EB%A0%A5.png) · [개선 후 재검증](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%98%A4%ED%94%84%EC%85%8B%20%EA%B2%B0%EA%B3%BC%EC%97%86%EC%9D%8C.png) |
| 최신순 | 작성일 DESC, 동점 ID DESC로 정렬합니다. | 원본의 날짜 증가 오류를 수정했습니다. 개선 후 10월 28일에서 ID 973→945→917을 확인했습니다. [전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%B5%9C%EC%8B%A0%EC%88%9C%EA%B2%80%EC%83%89.png) · [후](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%B5%9C%EC%8B%A0%EC%88%9C%20%EA%B2%80%EC%A6%9D.png) |
| 오래된순 | 작성일 ASC, 동점 ID ASC로 정렬합니다. | 개선 후 4월 1일에서 ID 20→48→76을 확인했습니다. [전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%98%A4%EB%9E%98%EB%90%9C%EC%88%9C%EA%B2%80%EC%83%89.png) · [후](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%98%A4%EB%9E%98%EB%90%9C%EC%88%9C%20%EA%B2%80%EC%A6%9D.png) |
| 조회수순·동점 | 조회수 DESC, 동점 ID DESC로 정렬합니다. | 캡처 시 4,969회에서 736→387, 4,956회에서 240→144를 확인했습니다. [기존](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%A1%B0%ED%9A%8C%EC%88%98%EC%88%9C%EA%B2%80%EC%83%89.png) · [동점 검증](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%A1%B0%ED%9A%8C%EC%88%98%EC%88%9C%EA%B2%80%EC%A6%9D.png) |
| `sort=invalid` | `latest`로 처리합니다. | 기본 최신순 기준과 비교하고 적용 정렬 키를 확인했습니다. [기존](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%ED%97%88%EC%9A%A9%EB%90%98%EC%A7%80%EC%95%8A%EC%9D%80%EC%9D%BC%EB%B0%98%EB%AC%B8%EC%9E%90%EC%97%B4.png) · [최신순 기준](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-%EC%B5%9C%EC%8B%A0%EC%88%9C%EA%B8%B0%EC%A4%80.png) · [invalid 결과](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-invalid%ED%85%8C%EC%8A%A4%ED%8A%B8.png) |
| `sort=__proto__` | 상속 속성을 차단하고 `latest`로 처리합니다. | 원본의 TypeError 대신 최신순 JSON이 반환됩니다. [오류](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%83%81%EC%86%8D%EC%86%8D%EC%84%B1%EC%98%A4%EB%A5%98.png) · [개선 결과](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%EA%B0%9C%EC%84%A0-proto%ED%85%8C%EC%8A%A4%ED%8A%B8.png) |
| `sort=constructor` | 상속 속성을 차단하고 `latest`로 처리합니다. | 최신순 기준과 같은 첫 ID 973의 정상 JSON을 확인했습니다. [결과](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%EA%B0%9C%EC%84%A0-constructor%ED%85%8C%EC%8A%A4%ED%8A%B8.png) |
| 입력 직후 조회·정렬 변경 | 중복 요청 없이 현재 검색어를 유지합니다. | 수동 조회 요청 1회와 정렬 변경 후 Redis 유지를 확인했습니다. [요청 1회](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EA%B2%80%EC%83%89%20%EC%9A%94%EC%B2%AD%201%ED%9A%8C.png) · [검색어 유지](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%A1%B0%ED%9A%8C%EC%88%98%20%EB%B3%80%EA%B2%BD%20%EC%8B%9C%20%EA%B2%80%EC%83%89%EC%96%B4%20%EC%9C%A0%EC%A7%80.png) |



### 4.2 페이지네이션 — 35점

| 검증 입력·행동 | 기대 결과 | 실제 결과·비교 근거 |
|---|---|---|
| 원본 `/api/posts?page=2` | 페이지 이동과 meta가 일치합니다. | ID 990~981, `page=2`, `limit=10`, `totalCount=1000`, `totalPages=100`, `hasPrev=true`를 확인했습니다. [2페이지](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/2%ED%8E%98%EC%9D%B4%EC%A7%80.png) |
| `page=0,-1,abc` | 모두 1페이지로 보정합니다. | 원본부터 제공된 방어 로직을 유지했습니다. [0](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/page%3D0%2Climit%3D1000%ED%85%8C%EC%8A%A4%ED%8A%B8.png) · [-1](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/-1%ED%8E%98%EC%9D%B4%EC%A7%80%EC%B2%98%EB%A6%AC.png) · [abc](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/abc%ED%8E%98%EC%9D%B4%EC%A7%80.png) |
| `limit=1000` | 오프셋 상한인 100으로 보정합니다. | `limit=100`, `totalPages=10`을 확인했습니다. [입력 보정](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/page%3D0%2Climit%3D1000%ED%85%8C%EC%8A%A4%ED%8A%B8.png) |
| 검색·정렬 변경 | 1페이지로 초기화합니다. | 2→1페이지 변경을 확인했습니다. [검색 전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EA%B2%80%EC%83%89%EC%96%B4%20%EB%B3%80%EA%B2%BD-1.png) · [검색 후](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EA%B2%80%EC%83%89%EC%96%B4%EB%B3%80%EA%B2%BD-2.png) · [정렬 전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%A0%95%EB%A0%AC%EB%B3%80%EA%B2%BD-1.png) · [정렬 후](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%A0%95%EB%A0%AC%EB%B3%80%EA%B2%BD-2.png) |
| 커서 Redis 검색 | 제목 검색 후 ID DESC로 반환합니다. | 원본의 검색 누락을 수정했습니다. 첫 묶음 `nextCursor=883`, `hasNext=true`를 확인했습니다. [전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EA%B8%B0%EB%B0%98%EA%B2%80%EC%83%89.png) · [API 결과](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9CAPI%20%EC%A0%9C%EB%AA%A9%EA%B2%80%EC%83%89.png) |
| 커서 본문 검색 | 제목 외에 본문도 검색합니다. | 제목에 없는 본문 검색 결과와 실제 요청 URL을 확인했습니다. [결과](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%B3%B8%EB%AC%B8%EA%B2%80%EC%83%89.png) · [Network](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%B3%B8%EB%AC%B8%20%EA%B2%80%EC%83%89.png) |
| 커서 고정 정렬 | ID DESC 기준을 명시하고 선택을 막습니다. | 고정 옵션과 비활성화된 정렬 선택창을 확인했습니다. [개선 전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EC%A0%95%EB%A0%AC%EA%B8%B0%EC%A4%80%ED%99%95%EC%9D%B8.png) · [고정 UI](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20ID%20%EB%82%B4%EB%A6%BC%EC%B0%A8%EC%88%9C%20%EA%B3%A0%EC%A0%95.png) |
| `cursor=991&limit=10` | 991 미만의 다음 10건을 반환합니다. | ID 990~981, `nextCursor=981`을 확인했습니다. [다음 묶음](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EB%AA%A9%EB%A1%9D%EC%A1%B0%ED%9A%8C.png) |
| `cursor=11&limit=10` | 마지막 10건에서 다음 결과가 없습니다. | ID 10~1, `hasNext=false`를 확인했습니다. [종료 조건](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EC%A2%85%EB%A3%8C%EC%A1%B0%EA%B1%B4.png) |
| Redis 마지막까지 더 보기 | 누락·중복 없이 전체를 누적합니다. | 화면 83건·전체 83건·고유 ID 83개·전체 ID 순서 일치 `true`를 확인했습니다. [전체 대조](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%88%84%EC%A0%81%EB%AA%A9%EB%A1%9D%20%EB%88%84%EB%9D%BD%2C%EC%A4%91%EB%B3%B5%20%ED%99%95%EC%9D%B8.png) |
| 마지막 Redis 응답·버튼 | `hasNext=false`이고 더 보기가 비활성화됩니다. | `cursor=43`, `nextCursor=7`, 마지막 ID 31·19·7과 종료 버튼을 확인했습니다. [응답](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EA%B2%80%EC%83%89%20%EA%B2%80%EC%A6%9D.png) · [버튼](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%A7%88%EC%A7%80%EB%A7%89%20%EA%B2%80%EC%83%89%20%EB%8D%94%EB%B3%B4%EA%B8%B0%EB%B2%84%ED%8A%BC%20%EB%B9%84%ED%99%9C%EC%84%B1%ED%99%94.png) |
| 마지막에서 MVCC로 변경 | 기존 목록·커서를 초기화합니다. | 첫 ID 993의 10건, 요청의 `cursor=null`, `hasNext=true`, 버튼 재활성화를 확인했습니다. [초기화](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EA%B2%80%EC%83%89%EC%96%B4%EB%B3%80%EA%B2%BD%20%EC%8B%9C%20%EB%AA%A9%EB%A1%9D%EA%B3%BC%20%EC%BB%A4%EC%84%9C%EC%B4%88%EA%B8%B0%ED%99%94%20%ED%99%95%EC%9D%B8.png) |
| 없는 검색어 | 빈 결과와 종료 상태를 반환합니다. | 원본의 검색 조건 무시를 수정했습니다. `items=[]`, `nextCursor=null`, `hasNext=false`를 확인했습니다. [전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9CAPI%EA%B2%80%EC%83%89%ED%85%8C%EC%8A%A4%ED%8A%B8.png) · [후 API](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9CAPI%20%EC%97%86%EB%8A%94%EA%B2%80%EC%83%89%EC%96%B4.png) · [화면](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EC%97%86%EB%8A%94%EA%B2%80%EC%83%89%EC%96%B4.png) |

오프셋 2페이지의 ID 990~981은 **작성일 정렬 개선 전**의 검증 값입니다. 개선 후 최신순에서는 작성일 순서에 따라 목록이 달라집니다. 커서는 작성일과 관계없이 ID 내림차순이므로 작성일이 섞여 보일 수 있습니다.

Redis 누락·중복 검증은 오프셋 API에서 받은 전체 Redis 결과를 ID 내림차순으로 정렬하여 화면의 누적 ID와 대조했습니다. 커서 API 자체에는 `totalCount`가 없으므로 83건은 이 대조 과정에서 확인한 값입니다. 없는 검색어의 전후 사진은 서로 다른 문자열을 사용했으며, 모두 데이터에 없는 입력으로 검색 조건 적용 여부를 확인한 사례입니다.

### 4.3 비동기 처리 — 30점

| 검증 입력·행동 | 기대 결과 | 실제 결과·비교 근거 |
|---|---|---|
| No throttling에서 Redis를 300ms 미만 간격으로 입력 | 입력이 멈춘 뒤 검색 요청 1회가 발생합니다. | Network의 검색 요청 1건·200 응답을 확인했습니다. [기존 검증](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/300ms%EB%94%94%EB%B0%94%EC%9A%B4%EC%8A%A4.png) · [개선 후 재검증](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/300ms%20%EB%94%94%EB%B0%94%EC%9A%B4%EC%8A%A4%20%EC%9C%A0%EC%A7%80%20%ED%99%95%EC%9D%B8.png) |
| 3G에서 Redis 요청 중 MVCC로 변경 | 이전 요청을 취소하고 마지막 결과를 표시합니다. | 이전 Redis 요청은 canceled, 최종 MVCC 요청은 200이며 화면도 MVCC 결과입니다. [기존 검증](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/AbortController%EC%9A%94%EC%B2%AD%EC%B7%A8%EC%86%8C.png) · [개선 후 결과](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/%EC%9A%94%EC%B2%AD%20%EC%A7%84%ED%96%89%EC%A4%91%EC%9D%BC%20%EB%95%8C%20%EA%B2%80%EC%83%89%EC%96%B4%EB%A5%BC%20%EB%B0%94%EA%BE%B8%EB%A9%B4%20%EC%9D%B4%EC%A0%84%EC%9A%94%EC%B2%AD%EC%9D%B4%20%EC%B7%A8%EC%86%8C%EB%90%98%EB%8A%94%EC%A7%80%20%ED%99%95%EC%9D%B8.png) |
| `AbortError` 발생 | 정상 취소로 처리합니다. | Console의 정상 취소 로그를 확인했습니다. [Console](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/AbortController%EC%9D%B4%EC%A0%84%EB%B9%84%EB%8F%99%EA%B8%B0%EC%9A%94%EC%B2%AD%EC%B7%A8%EC%86%8C.png) |
| 취소 뒤 늦게 응답 처리 | 취소된 결과가 화면을 덮어쓰지 않습니다. | JSON 파싱 뒤 `signal.aborted` 검사로 반영을 차단합니다. [입력 시 취소](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/code/%EC%9D%B4%EC%A0%84%EC%9A%94%EC%B2%AD%EC%B7%A8%EC%86%8C.png) · [반영 차단](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/code/%EC%B7%A8%EC%86%8C%EB%90%9C%EC%9A%94%EC%B2%AD%EC%9D%98%20%EC%9D%91%EB%8B%B5%20%EB%B0%98%EC%98%81%EC%95%88%ED%95%A8.png) |

디바운스 요청 1회 실험과 3G 취소 실험은 별도입니다. 3G 사진의 중간 입력 요청 여러 건을 디바운스 실패로 해석하지 않습니다. 화면의 `3.00ms` 같은 배지는 응답 소요 시간이며 300ms 디바운스 대기를 측정한 값이 아닙니다.

### 4.4 URL 상태 복원 — 10점

| 검증 입력·행동 | 기대 결과 | 실제 결과·비교 근거 |
|---|---|---|
| 오프셋 검색·정렬·페이지 변경 | `URLSearchParams`로 주소와 화면을 동기화합니다. | 사용자 이동은 `pushState`, 응답 보정은 `replaceState`로 처리합니다. [동기화 코드](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/code/%EC%9E%98%EB%AA%BB%EB%90%9C%20URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90.png) |
| MVCC·조회수순·2페이지에서 새로고침 | 검색어·정렬·페이지·결과를 유지합니다. | 전후 첫 ID 777과 목록이 일치합니다. [전](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EC%83%88%EB%A1%9C%EA%B3%A0%EC%B9%A8%20%EC%9D%B4%EC%A0%84.png) · [후](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EC%83%88%EB%A1%9C%EA%B3%A0%EC%B9%A8%20%EC%9D%B4%ED%9B%84.png) |
| Redis·오래된순·1페이지에서 뒤로 가기 | 직전 상태를 복원합니다. | Redis·조회수순·1페이지·첫 ID 487을 확인했습니다. [이동 전](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%EC%9D%B4%EC%A0%84.png) · [뒤로 가기](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%EC%9D%B4%ED%9B%84.png) |
| 한 번 더 뒤로 가기 | 이전 검색어·정렬·페이지를 복원합니다. | URL의 `page=2&sort=views&search=MVCC`와 입력창·선택창·목록이 일치합니다. [URL과 결과](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%EC%9D%B4%ED%9B%84-2.png) |
| `?page=9999&sort=invalid` | 서버의 안전한 페이지·정렬로 보정합니다. | 주소가 `page=100&limit=10&sort=latest`로 바뀌고 최신순·100페이지가 표시됩니다. [전](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/URL%20%EC%83%81%ED%83%9C%20%EB%B3%B5%EC%9B%90/%EC%9E%98%EB%AA%BB%EB%90%9C%20URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90%20%ED%85%8C%EC%8A%A4%ED%8A%B8.png) · [후](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EC%9E%98%EB%AA%BB%EB%90%9C%20URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90.png) |

최초 로드와 `popstate`는 공통 `restoreFromUrl()`을 호출합니다. 복원 전에 예약 검색과 진행 중인 요청을 취소하며, 복원 조회는 `loadData(false)`로 실행하여 새 히스토리 기록을 추가하지 않습니다. [이벤트 연결](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/code/%EC%83%88%EB%A1%9C%EA%B3%A0%EC%B9%A8%2C%20%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%ED%9B%84%20%EA%B2%B0%EA%B3%BC%EC%9D%BC%EC%B9%98.png)

새로고침 비교 사진은 입력·정렬·페이지·목록의 유지를 보여줍니다. 주소창이 포함된 뒤로 가기 사진은 URL과 화면의 일치를 보여줍니다. 잘못된 URL의 전후 사진은 주소 보정 개선 단계의 기록입니다.

<a id="evidence"></a>

## 7. 전체 증거 사진 — 74장

`evidence`의 사진 74장을 모두 연결했습니다. 파일명 링크를 누르면 원본 사진을 열 수 있습니다. 모든 사진이 최종 완성 화면인 것은 아니므로 아래 설명에서 기존 동작·오류·수정 코드·중간 단계·재검증을 구분합니다.

### 기능검증 — 검색·정렬 — 9장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| B-S01 | [화면진입.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%ED%99%94%EB%A9%B4%EC%A7%84%EC%9E%85.png) | 첨부 예제의 초기 게시판 화면을 확인합니다. |
| B-S02 | [제목검색.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%A0%9C%EB%AA%A9%EA%B2%80%EC%83%89.png) | Redis 제목 검색의 기존 동작을 확인합니다. |
| B-S03 | [없는검색어입력.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%97%86%EB%8A%94%EA%B2%80%EC%83%89%EC%96%B4%EC%9E%85%EB%A0%A5.png) | 오프셋 모드에서 검색 결과가 없을 때의 안내를 확인합니다. |
| B-S04 | [최신순검색.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%B5%9C%EC%8B%A0%EC%88%9C%EA%B2%80%EC%83%89.png) | 최신순이 ID만 감소시키고 작성일은 증가하는 오류를 확인합니다. |
| B-S05 | [오래된순검색.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%98%A4%EB%9E%98%EB%90%9C%EC%88%9C%EA%B2%80%EC%83%89.png) | 오래된순의 작성일이 선택한 방향과 다른 오류를 확인합니다. |
| B-S06 | [조회수순검색.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%A1%B0%ED%9A%8C%EC%88%98%EC%88%9C%EA%B2%80%EC%83%89.png) | 기존 조회수 내림차순 정렬을 확인합니다. |
| B-S07 | [허용되지않은일반문자열.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%ED%97%88%EC%9A%A9%EB%90%98%EC%A7%80%EC%95%8A%EC%9D%80%EC%9D%BC%EB%B0%98%EB%AC%B8%EC%9E%90%EC%97%B4.png) | 허용되지 않은 일반 정렬 키의 기본값 처리를 확인합니다. |
| B-S08 | [상속속성.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%83%81%EC%86%8D%EC%86%8D%EC%84%B1.png) | 초기에 sort=proto로 요청한 기록입니다. 일반 문자열 검사이며 상속 속성 검증 증거와 구분합니다. |
| B-S09 | [상속속성오류.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/%EC%83%81%EC%86%8D%EC%86%8D%EC%84%B1%EC%98%A4%EB%A5%98.png) | 실제 `sort=__proto__` 요청에서 발생한 TypeError를 확인합니다. |

### 기능검증 — 페이지네이션 — 13장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| B-P01 | [2페이지.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/2%ED%8E%98%EC%9D%B4%EC%A7%80.png) | 원본 오프셋 2페이지 목록과 meta 응답을 확인합니다. |
| B-P02 | [-1페이지처리.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/-1%ED%8E%98%EC%9D%B4%EC%A7%80%EC%B2%98%EB%A6%AC.png) | page=-1이 1페이지로 보정되는지 확인합니다. |
| B-P03 | [abc페이지.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/abc%ED%8E%98%EC%9D%B4%EC%A7%80.png) | page=abc가 1페이지로 보정되는지 확인합니다. |
| B-P04 | [page=0,limit=1000테스트.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/page%3D0%2Climit%3D1000%ED%85%8C%EC%8A%A4%ED%8A%B8.png) | page=0은 1, limit=1000은 100으로 보정되는지 확인합니다. |
| B-P05 | [검색어 변경-1.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EA%B2%80%EC%83%89%EC%96%B4%20%EB%B3%80%EA%B2%BD-1.png) | 검색어를 변경하기 전 페이지 상태를 기록합니다. |
| B-P06 | [검색어변경-2.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EA%B2%80%EC%83%89%EC%96%B4%EB%B3%80%EA%B2%BD-2.png) | 검색어 변경 후 1페이지로 초기화되는지 확인합니다. |
| B-P07 | [정렬변경-1.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%A0%95%EB%A0%AC%EB%B3%80%EA%B2%BD-1.png) | 정렬을 변경하기 전 페이지 상태를 기록합니다. |
| B-P08 | [정렬변경-2.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%A0%95%EB%A0%AC%EB%B3%80%EA%B2%BD-2.png) | 정렬 변경 후 1페이지로 초기화되는지 확인합니다. |
| B-P09 | [커서목록조회.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EB%AA%A9%EB%A1%9D%EC%A1%B0%ED%9A%8C.png) | 원본 커서 API의 다음 묶음과 nextCursor를 확인합니다. |
| B-P10 | [커서기반검색.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EA%B8%B0%EB%B0%98%EA%B2%80%EC%83%89.png) | 검색창의 Redis가 커서 요청에 전달되지 않는 문제를 확인합니다. |
| B-P11 | [커서정렬기준확인.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EC%A0%95%EB%A0%AC%EA%B8%B0%EC%A4%80%ED%99%95%EC%9D%B8.png) | 개선 전 커서 모드의 표시와 정렬 선택 상태를 기록합니다. |
| B-P12 | [커서종료조건.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9C%EC%A2%85%EB%A3%8C%EC%A1%B0%EA%B1%B4.png) | 마지막 커서 묶음에서 hasNext=false가 반환되는지 확인합니다. |
| B-P13 | [커서API검색테스트.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/%EC%BB%A4%EC%84%9CAPI%EA%B2%80%EC%83%89%ED%85%8C%EC%8A%A4%ED%8A%B8.png) | 없는 검색어를 전달해도 목록이 반환되는 원본 커서 검색 누락을 확인합니다. |

### 기능검증 — 비동기 처리 — 3장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| B-A01 | [300ms디바운스.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/300ms%EB%94%94%EB%B0%94%EC%9A%B4%EC%8A%A4.png) | 기존 300ms 디바운스의 Network 요청을 확인합니다. |
| B-A02 | [AbortController요청취소.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/AbortController%EC%9A%94%EC%B2%AD%EC%B7%A8%EC%86%8C.png) | 느린 네트워크에서 이전 요청의 취소와 마지막 검색 결과를 확인합니다. |
| B-A03 | [AbortController이전비동기요청취소.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/AbortController%EC%9D%B4%EC%A0%84%EB%B9%84%EB%8F%99%EA%B8%B0%EC%9A%94%EC%B2%AD%EC%B7%A8%EC%86%8C.png) | AbortError를 정상 취소로 처리한 Console 로그를 확인합니다. |

### 기능검증 — URL 상태 복원 — 4장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| B-U01 | [URL상태복원-1.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/URL%20%EC%83%81%ED%83%9C%20%EB%B3%B5%EC%9B%90/URL%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90-1.png) | 기존 URL 상태 복원 실험의 첫 상태를 기록합니다. |
| B-U02 | [URL상태복원-2.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/URL%20%EC%83%81%ED%83%9C%20%EB%B3%B5%EC%9B%90/URL%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90-2.png) | 기존 URL 상태 복원 실험의 변경 상태를 기록합니다. |
| B-U03 | [URL상태복원-3(뒤로가기).png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/URL%20%EC%83%81%ED%83%9C%20%EB%B3%B5%EC%9B%90/URL%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90-3%28%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%29.png) | 뒤로 가기 후의 기존 복원 동작을 확인합니다. |
| B-U04 | [잘못된 URL 상태복원 테스트.png](evidence/%EA%B8%B0%EB%8A%A5%EA%B2%80%EC%A6%9D/URL%20%EC%83%81%ED%83%9C%20%EB%B3%B5%EC%9B%90/%EC%9E%98%EB%AA%BB%EB%90%9C%20URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90%20%ED%85%8C%EC%8A%A4%ED%8A%B8.png) | URL 보정 전 주소·정렬 선택·실제 페이지의 불일치를 확인합니다. |

### 코드개선 — 검색·정렬 / code — 5장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-SC01 | [최신순,오래된순,조회수순 검증.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%B5%9C%EC%8B%A0%EC%88%9C%2C%EC%98%A4%EB%9E%98%EB%90%9C%EC%88%9C%2C%EC%A1%B0%ED%9A%8C%EC%88%98%EC%88%9C%20%EA%B2%80%EC%A6%9D.png) | 작성일 우선 비교와 동점 ID 비교의 변경 내용을 확인합니다. |
| A-SC02 | [정렬키 화이트리스트 개선-1.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%A0%95%EB%A0%AC%ED%82%A4%20%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-1.png) | 문자열 검사와 Object.hasOwn을 사용한 정렬 키 검사를 확인합니다. |
| A-SC03 | [정렬키 화이트리스트 개선-2.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%A0%95%EB%A0%AC%ED%82%A4%20%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-2.png) | 오프셋 meta에 적용 정렬 키 sort를 추가한 내용을 확인합니다. |
| A-SC04 | [검색어 유지하며 조회.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EA%B2%80%EC%83%89%EC%96%B4%20%EC%9C%A0%EC%A7%80%ED%95%98%EB%A9%B0%20%EC%A1%B0%ED%9A%8C.png) | 정렬 변경 시 공통 applySearch를 호출하는 코드를 확인합니다. |
| A-SC05 | [조회 시 중복요청 방지.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/code/%EC%A1%B0%ED%9A%8C%20%EC%8B%9C%20%EC%A4%91%EB%B3%B5%EC%9A%94%EC%B2%AD%20%EB%B0%A9%EC%A7%80.png) | applySearch에서 예약 검색을 취소하고 현재 입력을 읽는 코드를 확인합니다. |

### 코드개선 — 검색·정렬 / photo — 13장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-SP01 | [제목검색.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%A0%9C%EB%AA%A9%EA%B2%80%EC%83%89.png) | Redis 제목 검색에 작성일 우선 정렬이 적용되는지 확인합니다. |
| A-SP02 | [본문 검색 결과.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EB%B3%B8%EB%AC%B8%20%EA%B2%80%EC%83%89%20%EA%B2%B0%EA%B3%BC.png) | 제목에 없는 상세 본문 문구가 JSON content에 포함되는지 확인합니다. |
| A-SP03 | [본문 검색.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EB%B3%B8%EB%AC%B8%20%EA%B2%80%EC%83%89.png) | 본문 검색의 totalCount=1000과 totalPages=100을 확인합니다. |
| A-SP04 | [오프셋 결과없음.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%98%A4%ED%94%84%EC%85%8B%20%EA%B2%B0%EA%B3%BC%EC%97%86%EC%9D%8C.png) | 오프셋 빈 목록 안내와 items=[], totalCount=0을 확인합니다. |
| A-SP05 | [최신순 검증.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%B5%9C%EC%8B%A0%EC%88%9C%20%EA%B2%80%EC%A6%9D.png) | 작성일 내림차순과 같은 날짜의 ID 내림차순을 확인합니다. |
| A-SP06 | [오래된순 검증.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%98%A4%EB%9E%98%EB%90%9C%EC%88%9C%20%EA%B2%80%EC%A6%9D.png) | 작성일 오름차순과 같은 날짜의 ID 오름차순을 확인합니다. |
| A-SP07 | [조회수순검증.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%A1%B0%ED%9A%8C%EC%88%98%EC%88%9C%EA%B2%80%EC%A6%9D.png) | 조회수 내림차순과 조회수 동점의 ID 내림차순을 확인합니다. |
| A-SP08 | [화이트리스트 개선-최신순기준.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-%EC%B5%9C%EC%8B%A0%EC%88%9C%EA%B8%B0%EC%A4%80.png) | 잘못된 정렬 키의 결과와 비교할 최신순 기준 응답을 기록합니다. |
| A-SP09 | [화이트리스트 개선-invalid테스트.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%20%EA%B0%9C%EC%84%A0-invalid%ED%85%8C%EC%8A%A4%ED%8A%B8.png) | sort=invalid 요청에서 최신순 기본값이 적용되는지 확인합니다. |
| A-SP10 | [화이트리스트개선-proto테스트.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%EA%B0%9C%EC%84%A0-proto%ED%85%8C%EC%8A%A4%ED%8A%B8.png) | 실제 `__proto__` 요청이 오류 없이 최신순으로 처리되는지 확인합니다. |
| A-SP11 | [화이트리스트개선-constructor테스트.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%ED%99%94%EC%9D%B4%ED%8A%B8%EB%A6%AC%EC%8A%A4%ED%8A%B8%EA%B0%9C%EC%84%A0-constructor%ED%85%8C%EC%8A%A4%ED%8A%B8.png) | constructor 요청이 최신순 기준과 같은 JSON을 반환하는지 확인합니다. |
| A-SP12 | [검색 요청 1회.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EA%B2%80%EC%83%89%20%EC%9A%94%EC%B2%AD%201%ED%9A%8C.png) | 입력 직후 조회할 때 예약 검색과 즉시 조회가 중복되지 않는지 확인합니다. |
| A-SP13 | [조회수 변경 시 검색어 유지.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EA%B2%80%EC%83%89%2C%EC%A0%95%EB%A0%AC/photo/%EC%A1%B0%ED%9A%8C%EC%88%98%20%EB%B3%80%EA%B2%BD%20%EC%8B%9C%20%EA%B2%80%EC%83%89%EC%96%B4%20%EC%9C%A0%EC%A7%80.png) | 정렬을 조회수순으로 변경해도 현재 검색어가 유지되는지 확인합니다. |

### 코드개선 — 페이지네이션 / code — 4장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-PC01 | [커서API 제목,본문 추가.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9CAPI%20%EC%A0%9C%EB%AA%A9%2C%EB%B3%B8%EB%AC%B8%20%EC%B6%94%EA%B0%80.png) | 커서 API에 제목·본문 검색과 커서 조건을 함께 적용한 코드를 확인합니다. |
| A-PC02 | [커서API 검색추가.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9CAPI%20%EA%B2%80%EC%83%89%EC%B6%94%EA%B0%80.png) | 프론트엔드 커서 요청의 search 파라미터 추가를 확인합니다. |
| A-PC03 | [커서모드 ID내림차순고정.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20ID%EB%82%B4%EB%A6%BC%EC%B0%A8%EC%88%9C%EA%B3%A0%EC%A0%95.png) | 커서 모드에서 정렬 선택을 비활성화하고 오프셋 선택을 복원하는 코드를 확인합니다. |
| A-PC04 | [커서모드 고정기준 표시.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/code/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EA%B3%A0%EC%A0%95%EA%B8%B0%EC%A4%80%20%ED%91%9C%EC%8B%9C.png) | ID 내림차순 고정을 표시하는 전용 옵션을 확인합니다. |

### 코드개선 — 페이지네이션 / photo — 9장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-PP01 | [커서API 제목검색.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9CAPI%20%EC%A0%9C%EB%AA%A9%EA%B2%80%EC%83%89.png) | Redis 검색 결과와 첫 nextCursor=883, hasNext=true를 확인합니다. |
| A-PP02 | [커서모드 Redis 검색.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20Redis%20%EA%B2%80%EC%83%89.png) | 커서 검색 추가 직후의 Redis 화면입니다. 정렬 UI 고정 전의 중간 단계로 구분합니다. |
| A-PP03 | [커서API 없는검색어.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9CAPI%20%EC%97%86%EB%8A%94%EA%B2%80%EC%83%89%EC%96%B4.png) | 없는 검색어에서 items=[], nextCursor=null, hasNext=false를 확인합니다. |
| A-PP04 | [커서모드 없는검색어.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EC%97%86%EB%8A%94%EA%B2%80%EC%83%89%EC%96%B4.png) | 빈 목록 안내와 끝 버튼을 확인합니다. 정렬 UI 고정 전의 중간 단계 화면입니다. |
| A-PP05 | [커서모드 ID 내림차순 고정.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20ID%20%EB%82%B4%EB%A6%BC%EC%B0%A8%EC%88%9C%20%EA%B3%A0%EC%A0%95.png) | 커서 모드의 ID 내림차순 표시와 정렬 선택 비활성화를 확인합니다. |
| A-PP06 | [커서모드 검색 검증.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EA%B2%80%EC%83%89%20%EA%B2%80%EC%A6%9D.png) | 마지막 Redis 응답의 cursor=43, nextCursor=7, hasNext=false를 확인합니다. |
| A-PP07 | [커서모드 누적목록 누락,중복 확인.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%88%84%EC%A0%81%EB%AA%A9%EB%A1%9D%20%EB%88%84%EB%9D%BD%2C%EC%A4%91%EB%B3%B5%20%ED%99%95%EC%9D%B8.png) | 화면 83건·전체 83건·고유 ID 83개와 전체 ID 순서 일치를 확인합니다. |
| A-PP08 | [커서모드 마지막 검색 더보기버튼 비활성화.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%A7%88%EC%A7%80%EB%A7%89%20%EA%B2%80%EC%83%89%20%EB%8D%94%EB%B3%B4%EA%B8%B0%EB%B2%84%ED%8A%BC%20%EB%B9%84%ED%99%9C%EC%84%B1%ED%99%94.png) | 마지막 ID 31·19·7과 더 이상 글이 없습니다 버튼의 종료 상태를 확인합니다. |
| A-PP09 | [검색어변경 시 목록과 커서초기화 확인.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%ED%8E%98%EC%9D%B4%EC%A7%80%EB%84%A4%EC%9D%B4%EC%85%98/photo/%EA%B2%80%EC%83%89%EC%96%B4%EB%B3%80%EA%B2%BD%20%EC%8B%9C%20%EB%AA%A9%EB%A1%9D%EA%B3%BC%20%EC%BB%A4%EC%84%9C%EC%B4%88%EA%B8%B0%ED%99%94%20%ED%99%95%EC%9D%B8.png) | MVCC 변경 후 누적 목록·커서 초기화와 더 보기 재활성화를 확인합니다. |

### 코드개선 — 비동기 처리 / code — 2장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-AC01 | [이전요청취소.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/code/%EC%9D%B4%EC%A0%84%EC%9A%94%EC%B2%AD%EC%B7%A8%EC%86%8C.png) | 입력 이벤트에서 진행 중인 이전 요청을 즉시 취소하는 코드를 확인합니다. |
| A-AC02 | [취소된요청의 응답 반영안함.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/code/%EC%B7%A8%EC%86%8C%EB%90%9C%EC%9A%94%EC%B2%AD%EC%9D%98%20%EC%9D%91%EB%8B%B5%20%EB%B0%98%EC%98%81%EC%95%88%ED%95%A8.png) | JSON 파싱 후 signal.aborted로 화면 반영을 차단하는 코드를 확인합니다. |

### 코드개선 — 비동기 처리 / photo — 4장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-AP01 | [300ms 디바운스 유지 확인.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/300ms%20%EB%94%94%EB%B0%94%EC%9A%B4%EC%8A%A4%20%EC%9C%A0%EC%A7%80%20%ED%99%95%EC%9D%B8.png) | 연속 입력 후 검색 요청 1건이 발생하는지 확인합니다. |
| A-AP02 | [요청 진행중일 때 검색어를 바꾸면 이전요청이 취소되는지 확인.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/%EC%9A%94%EC%B2%AD%20%EC%A7%84%ED%96%89%EC%A4%91%EC%9D%BC%20%EB%95%8C%20%EA%B2%80%EC%83%89%EC%96%B4%EB%A5%BC%20%EB%B0%94%EA%BE%B8%EB%A9%B4%20%EC%9D%B4%EC%A0%84%EC%9A%94%EC%B2%AD%EC%9D%B4%20%EC%B7%A8%EC%86%8C%EB%90%98%EB%8A%94%EC%A7%80%20%ED%99%95%EC%9D%B8.png) | 3G에서 Redis 요청 취소와 최종 MVCC 요청 완료를 확인합니다. |
| A-AP03 | [커서모드 본문검색.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%B3%B8%EB%AC%B8%EA%B2%80%EC%83%89.png) | 커서 본문 검색 결과와 ID 내림차순 고정 UI를 함께 확인합니다. |
| A-AP04 | [커서모드 본문 검색.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/%EB%B9%84%EB%8F%99%EA%B8%B0%EC%B2%98%EB%A6%AC/photo/%EC%BB%A4%EC%84%9C%EB%AA%A8%EB%93%9C%20%EB%B3%B8%EB%AC%B8%20%EA%B2%80%EC%83%89.png) | 커서 본문 검색 요청의 URL과 Network 응답을 확인합니다. |

### 코드개선 — URL 상태 복원 / code — 2장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-UC01 | [잘못된 URL 상태복원.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/code/%EC%9E%98%EB%AA%BB%EB%90%9C%20URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90.png) | 응답 meta의 보정값을 상태·입력창·URL에 반영하는 코드를 확인합니다. |
| A-UC02 | [새로고침, 뒤로가기 후 결과일치.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/code/%EC%83%88%EB%A1%9C%EA%B3%A0%EC%B9%A8%2C%20%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%ED%9B%84%20%EA%B2%B0%EA%B3%BC%EC%9D%BC%EC%B9%98.png) | DOMContentLoaded와 popstate가 공통 restoreFromUrl을 호출하는 코드를 확인합니다. |

### 코드개선 — URL 상태 복원 / photo — 6장

| 번호 | 사진 링크 | 확인 내용 |
|---|---|---|
| A-UP01 | [새로고침 이전.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EC%83%88%EB%A1%9C%EA%B3%A0%EC%B9%A8%20%EC%9D%B4%EC%A0%84.png) | MVCC·조회수순·2페이지의 새로고침 전 상태를 기록합니다. |
| A-UP02 | [새로고침 이후.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EC%83%88%EB%A1%9C%EA%B3%A0%EC%B9%A8%20%EC%9D%B4%ED%9B%84.png) | 새로고침 후에도 같은 입력·정렬·페이지·목록이 유지되는지 확인합니다. |
| A-UP03 | [뒤로가기 이전.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%EC%9D%B4%EC%A0%84.png) | 정렬을 오래된순으로 바꾼 Redis·1페이지 상태를 기록합니다. |
| A-UP04 | [뒤로가기 이후.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%EC%9D%B4%ED%9B%84.png) | 뒤로 가기 후 Redis·조회수순·1페이지 복원을 확인합니다. |
| A-UP05 | [뒤로가기 이후-2.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EB%92%A4%EB%A1%9C%EA%B0%80%EA%B8%B0%20%EC%9D%B4%ED%9B%84-2.png) | 한 번 더 뒤로 가서 MVCC·조회수순·2페이지와 URL의 일치를 확인합니다. |
| A-UP06 | [잘못된 URL 상태복원.png](evidence/%EC%BD%94%EB%93%9C%EA%B0%9C%EC%84%A0/URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90/photo/%EC%9E%98%EB%AA%BB%EB%90%9C%20URL%20%EC%83%81%ED%83%9C%EB%B3%B5%EC%9B%90.png) | 잘못된 주소를 page=100, sort=latest로 보정한 화면을 확인합니다. |

