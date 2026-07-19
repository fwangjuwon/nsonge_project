# Architecture 문서

## 1. 전체 구조

```mermaid
flowchart LR
    User["사용자 브라우저"]
    FO["Frontend\nVue 3 / Vite / Vuetify"]
    BO["Backend API\nSpring Boot"]
    DB["PostgreSQL"]
    Gemini["Gemini API\nPrompt Generation"]
    ImageAI["NanoBanana\nImage Generation"]
    Storage["Local / Mounted Image Storage\n/images"]

    User --> FO
    FO -->|/api JSON, multipart| BO
    BO --> DB
    BO --> Gemini
    BO --> ImageAI
    BO --> Storage
    FO -->|image URL| Storage
```

## 2. 런타임 구성

| 구성요소 | 역할 |
|---|---|
| Browser | 사용자가 접근하는 웹 UI |
| Frontend | 정적 파일로 배포되는 Vue 애플리케이션 |
| Backend | API, 인증, 권한, 비즈니스 로직 |
| PostgreSQL | 사용자, 카테고리, 태그, 이력, 매핑 데이터 저장 |
| Gemini API | 상품/인물 프롬프트 생성 |
| NanoBanana image model | 상품/인물/합성/텍스트 이미지 생성 |
| Image Storage | 생성 이미지 파일 저장 및 `/images/**` 공개 |

## 3. 논리 아키텍처

```mermaid
flowchart TB
    subgraph FO["creative-studio-fo"]
        Router["Vue Router"]
        Store["Pinia Store"]
        Views["Views"]
        ApiClient["Axios API Client"]
    end

    subgraph BO["creative-studio-bo"]
        Controllers["Controllers"]
        Services["Services"]
        Mappers["MyBatis Mappers"]
        Security["Spring Security / JWT"]
        Exception["GlobalExceptionHandler"]
    end

    Views --> Store
    Views --> ApiClient
    Router --> Views
    ApiClient --> Controllers
    Security --> Controllers
    Controllers --> Services
    Services --> Mappers
    Exception --> Controllers
```

## 4. Backend Layer

```mermaid
flowchart LR
    Controller["Controller\nRequest/Response"]
    Service["Service\nBusiness Logic"]
    Mapper["Mapper\nMyBatis Interface"]
    XML["Mapper XML\nSQL"]
    DB["PostgreSQL"]
    External["External AI APIs"]

    Controller --> Service
    Service --> Mapper
    Mapper --> XML
    XML --> DB
    Service --> External
```

### Controller

- `/api/**` 엔드포인트를 제공합니다.
- `@PreAuthorize`로 권한을 제어합니다.
- 요청 DTO 검증 후 Service를 호출합니다.
- 응답은 `CommonResponse<T>`로 반환합니다.

### Service

- 핵심 비즈니스 규칙을 담당합니다.
- 일일 한도 체크, 참조 무결성 검증, 외부 AI API 호출, 이력 저장을 수행합니다.
- 클래스 레벨 `@Transactional(readOnly = true)`와 CUD 메서드 `@Transactional` 패턴을 사용합니다.

### Mapper

- MyBatis Mapper 인터페이스와 XML 쿼리로 DB를 조회/수정합니다.
- DB FK 제약이 없으므로 삭제/수정 전 참조 여부를 Service에서 확인합니다.

## 5. Frontend Layer

```mermaid
flowchart TB
    Layout["Layouts\nDefault/Auth"]
    Views["Views\nHome/Login/Prompt/Manage"]
    Store["Stores\nauth, promptRestore"]
    Api["API Client\nAxios"]
    Components["Common Components"]

    Layout --> Views
    Views --> Components
    Views --> Store
    Views --> Api
```

### Layout

- `AuthLayout`: 로그인 전 화면
- `DefaultLayout`: 로그인 후 사이드바/앱바 포함 화면

### Store

- `auth`: JWT, 로그인 사용자, role 관리
- `promptRestore`: 이력에서 생성 화면 복원 시 사용

### API Client

- JWT 자동 주입
- 401 응답 시 인증 상태 초기화 및 로그인 화면 이동
- `VITE_API_BASE_URL` 기반 base URL 설정

## 6. 인증 흐름

```mermaid
sequenceDiagram
    participant U as User
    participant FO as Frontend
    participant BO as Backend
    participant DB as PostgreSQL

    U->>FO: ID/PW 입력
    FO->>BO: POST /api/auth/login
    BO->>DB: loginId로 사용자 조회
    DB-->>BO: User
    BO->>BO: 비밀번호 검증, JWT 생성
    BO-->>FO: accessToken
    FO->>FO: localStorage 저장
    FO->>BO: Authorization: Bearer token
```

## 7. NSIST SSO 흐름

```mermaid
sequenceDiagram
    participant NSIST as NSIST
    participant FO as Frontend
    participant BO as Backend
    participant DB as PostgreSQL

    NSIST->>FO: /auth/login?emp_cd=&org_cd= iframe 진입
    FO->>BO: GET /api/auth/nsist-login?empCd=&orgCd=
    BO->>DB: empCd/orgCd 매핑 조회
    DB-->>BO: 매핑된 user_id
    BO->>BO: JWT 생성
    BO-->>FO: accessToken
    FO->>FO: 로그인 상태 저장 후 홈 이동
```

## 8. 상품 프롬프트 생성 흐름

```mermaid
sequenceDiagram
    participant U as User
    participant FO as Frontend
    participant BO as Backend
    participant DB as PostgreSQL
    participant G as Gemini

    U->>FO: SCS 카테고리, 태그, 스타일 선택
    FO->>BO: POST /api/prompts/goods
    BO->>DB: 사용자/한도/카테고리/태그 검증
    BO->>G: 프롬프트 생성 요청
    G-->>BO: 영문 프롬프트
    BO->>DB: prompt_history 저장
    BO-->>FO: PromptHistoryRespDto
```

## 9. 이미지 생성 흐름

```mermaid
sequenceDiagram
    participant FO as Frontend
    participant BO as Backend
    participant AI as NanoBanana
    participant FS as Image Storage
    participant DB as PostgreSQL

    FO->>BO: multipart/form-data 이미지 생성 요청
    BO->>BO: MIME/확장자/용량 검증
    BO->>AI: 이미지 생성 요청
    AI-->>BO: 이미지 바이너리/결과
    BO->>FS: uuid 파일 저장
    BO->>DB: image_url 업데이트
    BO-->>FO: imageUrls[]
```

## 10. 데이터 모델 요약

```mermaid
erDiagram
    USER ||--o{ PROMPT_HISTORY : creates
    CATEGORY ||--o{ PROMPT_HISTORY : selected_for_product
    TAG_GROUP ||--o{ TAG : contains
    PROMPT_HISTORY ||--o{ GOODS_PROMPT_HISTORY_TAG : has
    TAG ||--o{ GOODS_PROMPT_HISTORY_TAG : selected
    HUMAN_PROMPT_STEP ||--o{ HUMAN_PROMPT_ATTR : contains
    HUMAN_PROMPT_ATTR ||--o{ HUMAN_PROMPT_OPTION : contains
    PROMPT_HISTORY ||--o{ HUMAN_PROMPT_HISTORY_OPTION : has
    HUMAN_PROMPT_OPTION ||--o{ HUMAN_PROMPT_HISTORY_OPTION : selected

    USER {
        bigint id PK
        varchar login_id
        varchar role
        varchar team_name
        int daily_limit
        char use_yn
    }

    CATEGORY {
        bigint id PK
        varchar name
        varchar category_type
        bigint parent_id
    }

    PROMPT_HISTORY {
        bigint id PK
        bigint user_id
        bigint category_id
        varchar prompt_type
        varchar prompt_style
        text prompt_text
        varchar image_url
    }
```

## 11. 배포 아키텍처

실제 운영 환경은 사내 Private Cloud 기반입니다.

```mermaid
flowchart TB
    User["User Browser"]
    FE["Private Cloud\nFrontend Hosting"]
    BE["Private Cloud\nBackend API"]
    SQL["PostgreSQL"]
    Secret["Internal Config / Secrets"]
    AR["Private Artifact Repository"]
    Build["Internal CI/CD"]
    Gitea["Private Gitea"]
    Gemini["Gemini API"]

    User --> FE
    FE --> BE
    BE --> SQL
    BE --> Secret
    BE --> Gemini
    Gitea --> Build
    Build --> AR
    AR --> BE
    Build --> FE
```

## 12. 보안 포인트

- JWT Stateless 인증
- `ADMIN > MANAGER > USER` 역할 계층
- Swagger UI는 운영 환경 노출 여부 점검 필요
- `/images/**`는 공개 경로입니다.
- 업로드 이미지는 MIME 타입, 확장자, 파일 크기를 검증합니다.
- 실제 시크릿은 `.env` 또는 Secret Manager로 관리하고 Git에 커밋하지 않습니다.

## 13. 확장 포인트

- Refresh Token 도입
- 이미지 파일을 로컬 디스크에서 Object Storage로 이전
- DB FK 제약 또는 정합성 검사 배치 도입
- AI 모델 교체 가능하도록 provider abstraction 강화
- 프론트 테스트 도구 도입
- 운영 모니터링/알림 대시보드 구성
