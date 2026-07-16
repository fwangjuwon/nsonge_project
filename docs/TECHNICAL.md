# 기술문서

## 1. 개요

NS Creative Studio는 프론트엔드와 백엔드가 분리된 웹 애플리케이션입니다.

- FO는 사용자가 카테고리, 태그, 인물 속성, 합성 조건을 선택할 수 있는 UI를 제공합니다.
- BO는 인증, 권한, DB 영속화, Gemini 프롬프트 생성, NanoBanana 이미지 생성, 이미지 파일 저장을 담당합니다.

## 2. 기술 스택

### Backend

| 구분 | 기술 |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.5.11 |
| Build | Gradle |
| DB | PostgreSQL |
| SQL Mapper | MyBatis |
| Pagination | PageHelper |
| Security | Spring Security, JWT |
| API Docs | springdoc-openapi / Swagger UI |
| AI API | Gemini API, NanoBanana image model |
| Container | Docker |

### Frontend

| 구분 | 기술 |
|---|---|
| Framework | Vue 3 |
| Build | Vite |
| UI | Vuetify |
| State | Pinia |
| Router | Vue Router |
| HTTP | Axios |
| Icons | Material Design Icons |
| Runtime | Nginx container |

## 3. 디렉터리 구조

### Backend

```text
creative-studio-bo/
├── db/
│   ├── schema.sql
│   └── data.sql
├── docs/
├── src/main/java/com/nsmall/ns_creative_studio_bo/
│   ├── common/
│   │   ├── config/
│   │   ├── exception/
│   │   ├── page/
│   │   ├── security/
│   │   └── util/
│   └── domain/
│       ├── auth/
│       ├── category/
│       ├── prompt/
│       ├── tag/
│       └── user/
└── src/main/resources/
    ├── application.yml
    ├── application-local.yml
    ├── application-dev.yml
    ├── application-prd.yml
    └── mybatis/mappers/
```

도메인 패키지는 대체로 아래 구조를 따릅니다.

```text
domain/{domain}/
├── controller/
├── service/
├── mapper/
├── dto/request/
├── dto/response/
├── entity/
└── {Domain}ErrorCode.java
```

### Frontend

```text
creative-studio-fo/
├── public/
├── src/
│   ├── api/
│   ├── assets/
│   ├── components/
│   ├── composables/
│   ├── layouts/
│   ├── plugins/
│   ├── router/
│   ├── stores/
│   └── views/
└── docs/
```

## 4. Backend 설계

### 공통 응답

모든 API는 `CommonResponse<T>` 형태를 사용합니다.

```json
{
  "code": 200,
  "message": "OK",
  "data": {}
}
```

### 인증/인가

- 로그인 성공 시 Access Token을 발급합니다.
- Access Token 유효시간은 1시간입니다.
- Refresh Token은 사용하지 않습니다.
- API 요청은 `Authorization: Bearer {token}` 헤더를 사용합니다.
- 권한 계층은 `ADMIN > MANAGER > USER`입니다.

### 트랜잭션

서비스 구현체는 클래스 레벨에서 `@Transactional(readOnly = true)`를 사용하고, 생성/수정/삭제 메서드에 `@Transactional`을 별도로 선언하는 패턴을 따릅니다.

```java
@Service
@RequiredArgsConstructor
@Transactional(readOnly = true)
public class UserServiceImpl implements UserService {

    @Transactional
    public UserRespDto createUser(UserCreateReqDto dto) {
        ...
    }
}
```

이 패턴은 Primary/Replica DB 환경에서 읽기/쓰기 라우팅이 흔들리지 않도록 하기 위한 설계입니다.

### DB 설계 원칙

- 모든 PK는 `BIGINT GENERATED ALWAYS AS IDENTITY`입니다.
- 실제 DB FK 제약은 만들지 않고, 서비스 레이어에서 참조 무결성을 검증합니다.
- 공통 감사 컬럼은 다음 네 개를 사용합니다.
  - `init_regi_pic`
  - `init_regi_dttm`
  - `final_rvis_pic`
  - `final_rvis_dttm`

## 5. Frontend 설계

### 라우팅

| 경로 | 화면 | 권한 |
|---|---|---|
| `/auth/login` | 로그인 | 비로그인 |
| `/` | 홈 | 인증 사용자 |
| `/prompt/new` | 프롬프트/이미지 생성 | 인증 사용자 |
| `/prompt/history` | 이력 조회 | 인증 사용자 |
| `/manage/categories` | 카테고리 관리 | `MANAGER`, `ADMIN` |
| `/manage/tags` | 태그 관리 | `MANAGER`, `ADMIN` |
| `/manage/human-prompts` | 인물 속성 관리 | `MANAGER`, `ADMIN` |
| `/manage/users` | 사용자 관리 | `ADMIN` |

### Axios

`src/api/index.js`에서 Axios 인스턴스를 만들고 다음을 처리합니다.

- `VITE_API_BASE_URL` 기반 API URL 설정
- JWT 토큰 자동 주입
- 인증 API를 제외한 401 응답 시 로그인 화면 이동

## 6. 환경 변수

### Backend

| 이름 | 설명 |
|---|---|
| `GEMINI_API_KEY` | Gemini API Key |
| `POSTGRES_HOST` | PostgreSQL host |
| `POSTGRES_PORT` | PostgreSQL port |
| `POSTGRES_DB` | DB name |
| `POSTGRES_USER` | DB user |
| `POSTGRES_PASSWORD` | DB password |
| `JWT_SECRET` | JWT 서명 키 |

### Frontend

| 이름 | 설명 |
|---|---|
| `VITE_API_BASE_URL` | 백엔드 API base URL |

## 7. 테스트

Backend는 JUnit 5 기반 테스트를 사용합니다.

주요 Suite:

- `AllServiceTests`
- `AllControllerTests`
- `AllUnitTests`
- `AllLayersTests`

Frontend는 현재 별도 테스트 프레임워크보다 Vite build 검증 중심입니다.

## 8. 운영 주의사항

- 이미지 생성 API는 외부 AI API 응답 시간에 영향을 받으므로 프론트 요청 timeout을 일반 API보다 길게 설정합니다.
- 생성 이미지는 서버 파일 시스템의 `image.storage-path`에 저장되고, DB에는 `/images/{uuid}` 상대 경로가 저장됩니다.
- `/images/**`는 인증 없이 접근 가능합니다.
- `prd` 환경에서는 Swagger UI 노출 여부를 반드시 확인해야 합니다.
- 일부 기존 소스/SQL 주석은 한글 인코딩 깨짐이 있으므로 정리가 필요합니다.
