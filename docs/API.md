# API 문서

## 1. 기본 정보

| 항목 | 내용 |
|---|---|
| Base URL | `http://localhost:8080/api` |
| 인증 | JWT Bearer Token |
| 응답 형식 | JSON |
| 공통 응답 | `{ code, message, data }` |
| Swagger UI | `http://localhost:8080/swagger-ui/index.html` |

## 2. 인증 헤더

로그인/SSO 등 공개 API를 제외한 대부분의 API는 Bearer Token이 필요합니다.

```http
Authorization: Bearer {accessToken}
```

## 3. 공통 응답

```json
{
  "code": 200,
  "message": "OK",
  "data": {}
}
```

에러 응답도 동일한 래퍼를 사용합니다.

```json
{
  "code": 400,
  "message": "요청 값이 올바르지 않습니다.",
  "data": null
}
```

## 4. 권한

| Role | 설명 |
|---|---|
| `ADMIN` | 전체 기능 접근 |
| `MANAGER` | 관리 기능 일부 접근 |
| `USER` | 기본 생성/조회 기능 접근 |

권한 계층은 `ADMIN > MANAGER > USER`입니다.

## 5. Auth API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/auth/login` | Public | 자체 로그인 |
| `GET` | `/auth/nsist-login?empCd=&orgCd=` | Public | NSIST iframe SSO 로그인 |

### POST /auth/login

요청:

```json
{
  "loginId": "admin",
  "password": "password"
}
```

응답 data:

```json
{
  "accessToken": "...",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

## 6. User API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/users` | `ADMIN` | 사용자 생성 |
| `GET` | `/users` | `ADMIN` | 사용자 목록 조회 |
| `GET` | `/users/{id}` | `ADMIN` | 사용자 단건 조회 |
| `PUT` | `/users/{id}` | `ADMIN` | 사용자 수정, ban/unban |
| `DELETE` | `/users/{id}` | `ADMIN` | 사용자 삭제 |
| `PATCH` | `/users/me/password` | `USER+` | 본인 비밀번호 변경 |
| `GET` | `/users/me/usage` | `USER+` | 본인 금일 사용량 조회 |
| `GET` | `/users/{id}/usage` | `ADMIN` | 특정 사용자 사용량 조회 |
| `GET` | `/users/usage` | `ADMIN` | 전체 사용자 사용량 조회 |

### 사용자 목록 필터

`GET /users?name=&teamName=&role=`

| Query | 설명 |
|---|---|
| `name` | 이름 검색 |
| `teamName` | 팀명 검색 |
| `role` | `ADMIN`, `MANAGER`, `USER` |

## 7. Category API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/categories` | `MANAGER+` | 카테고리 생성 |
| `GET` | `/categories` | `USER+` | 카테고리 목록 조회 |
| `GET` | `/categories/{id}` | `USER+` | 카테고리 단건 조회 |
| `PUT` | `/categories/{id}` | `MANAGER+` | 카테고리 수정 |
| `DELETE` | `/categories/{id}` | `MANAGER+` | 카테고리 삭제 |

### 목록 조회

`GET /categories?parentId={id}`

- `parentId` 없음: LCS 목록
- `parentId` 있음: 해당 부모의 하위 카테고리 목록

카테고리 타입:

- `LCS`: 대분류
- `MCS`: 중분류
- `SCS`: 소분류, 프롬프트 생성 최종 선택값

## 8. Tag Group API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/tag-groups` | `MANAGER+` | 태그 그룹 생성 |
| `GET` | `/tag-groups` | `USER+` | 태그 그룹 목록 조회 |
| `GET` | `/tag-groups/{id}` | `USER+` | 태그 그룹 단건 조회 |
| `PUT` | `/tag-groups/{id}` | `MANAGER+` | 태그 그룹 수정 |
| `DELETE` | `/tag-groups/{id}` | `MANAGER+` | 태그 그룹 삭제 |

## 9. Tag API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/tags` | `MANAGER+` | 태그 생성 |
| `GET` | `/tags` | `USER+` | 태그 목록 조회 |
| `GET` | `/tags/{id}` | `USER+` | 태그 단건 조회 |
| `PUT` | `/tags/{id}` | `MANAGER+` | 태그 수정 |
| `DELETE` | `/tags/{id}` | `MANAGER+` | 태그 삭제 |

### 목록 조회

`GET /tags?tagGroupId={id}`

`tagGroupId`를 전달하면 특정 그룹의 태그만 조회합니다.

## 10. Product Prompt API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/prompts/goods` | `USER+` | 상품 프롬프트 생성 |
| `POST` | `/prompts/goods/generate-image` | `USER+` | 상품 이미지 생성 |
| `GET` | `/prompts` | `USER+` | 프롬프트 이력 조회 |
| `DELETE` | `/prompts/{id}` | `ADMIN` | 프롬프트 이력 삭제 |

### POST /prompts/goods

요청:

```json
{
  "goodsCategoryId": 1,
  "tagIds": [1, 2, 3],
  "imageAspectRatio": "1:1",
  "promptStyle": "NANO_BANANA",
  "purpose": "SNS 광고용",
  "additionalNotes": "밝은 스튜디오 배경",
  "textContent": "기능성 냉감 팬츠",
  "disclaimerText": "※ 예시 문구"
}
```

응답 data:

```json
{
  "id": 100,
  "promptType": "PRODUCT",
  "promptStyle": "NANO_BANANA",
  "promptText": "...",
  "imageAspectRatio": "1:1"
}
```

### POST /prompts/goods/generate-image

`multipart/form-data`

| Field | 필수 | 설명 |
|---|---|---|
| `historyId` | N | 이력 ID |
| `promptText` | Y | 이미지 생성 프롬프트 |
| `referenceImages` | N | 참조 이미지 목록 |

응답 data:

```json
{
  "imageUrls": [
    "http://localhost:8080/images/uuid-1.jpg",
    "http://localhost:8080/images/uuid-2.jpg"
  ]
}
```

## 11. Human Prompt API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `GET` | `/human-prompts/steps` | `USER+` | 인물 프롬프트 단계/속성/옵션 조회 |
| `POST` | `/human-prompts` | `USER+` | 인물 프롬프트 생성 |
| `POST` | `/human-prompts/generate-image` | `USER+` | 인물 이미지 생성 |
| `POST` | `/human-prompts/steps` | `MANAGER+` | Step 생성 |
| `PUT` | `/human-prompts/steps/{id}` | `MANAGER+` | Step 수정 |
| `DELETE` | `/human-prompts/steps/{id}` | `MANAGER+` | Step 삭제 |
| `POST` | `/human-prompts/attrs` | `MANAGER+` | Attr 생성 |
| `PUT` | `/human-prompts/attrs/{id}` | `MANAGER+` | Attr 수정 |
| `DELETE` | `/human-prompts/attrs/{id}` | `MANAGER+` | Attr 삭제 |
| `POST` | `/human-prompts/options` | `MANAGER+` | Option 생성 |
| `PUT` | `/human-prompts/options/{id}` | `MANAGER+` | Option 수정 |
| `DELETE` | `/human-prompts/options/{id}` | `MANAGER+` | Option 삭제 |

### POST /human-prompts

인물 프롬프트는 현재 인원별 선택 구조를 사용합니다.

요청 예시:

```json
{
  "persons": [
    {
      "selections": {
        "1": [10],
        "2": [20, 21]
      }
    }
  ],
  "commonSelections": {
    "7": [70]
  },
  "isGroup": false,
  "relationship": "",
  "additionalNotes": "밝고 자연스러운 표정",
  "imageAspectRatio": "1:1",
  "promptStyle": "NANO_BANANA"
}
```

## 12. Fusion Image API

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/prompts/fusion/generate-image` | `USER+` | 상품+인물 합성 이미지 생성 |
| `POST` | `/prompts/fusion/text-overlay` | `USER+` | 기존 이미지에 텍스트 오버레이 생성 |

### POST /prompts/fusion/generate-image

`multipart/form-data`

| Field | 필수 | 설명 |
|---|---|---|
| `productImage` | Y | 상품 이미지 |
| `personImage` | Y | 인물 이미지 |
| `pose` | N | 포즈/행동 설명 |
| `instruction` | N | 추가 지시사항 |
| `imageAspectRatio` | N | 이미지 비율 |
| `textContent` | N | 오버레이 문구 |
| `disclaimerText` | N | 단서 문구 |

### POST /prompts/fusion/text-overlay

`multipart/form-data`

| Field | 필수 | 설명 |
|---|---|---|
| `image` | Y | 기준 이미지 |
| `textContent` | N | 오버레이 문구 |
| `disclaimerText` | N | 단서 문구 |

## 13. Prompt History API

### GET /prompts

Query:

| Query | 설명 |
|---|---|
| `pageNum` | 페이지 번호, 기본 1 |
| `pageSize` | 페이지 크기, 기본 10 |
| `loginId` | 생성자 필터 |
| `promptType` | `PRODUCT`, `PERSON`, `FUSION`, `TEXT_OVL` |
| `promptStyle` | `NANO_BANANA`, `MID_JOURNEY` |
| `dateFrom` | 생성일 시작, `yyyy-MM-dd` |
| `dateTo` | 생성일 종료, `yyyy-MM-dd` |

응답 data:

```json
{
  "pageNum": 1,
  "pageSize": 10,
  "total": 100,
  "pages": 10,
  "items": [],
  "hasPreviousPage": false,
  "hasNextPage": true
}
```

## 14. NSIST Mapping API

### Employee Mapping

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/nsist-emp-mappings` | `ADMIN` | 사번 매핑 생성 |
| `GET` | `/nsist-emp-mappings` | `ADMIN` | 사번 매핑 목록 |
| `DELETE` | `/nsist-emp-mappings/{empCd}` | `ADMIN` | 사번 매핑 삭제 |

### Organization Mapping

| Method | Path | 권한 | 설명 |
|---|---|---|---|
| `POST` | `/nsist-org-mappings` | `ADMIN` | 조직 매핑 생성 |
| `GET` | `/nsist-org-mappings` | `ADMIN` | 조직 매핑 목록 |
| `DELETE` | `/nsist-org-mappings/{orgCd}` | `ADMIN` | 조직 매핑 삭제 |

## 15. 파일 업로드 제약

이미지 업로드는 다음 형식을 허용합니다.

- `jpg`
- `jpeg`
- `png`
- `webp`

파일 크기는 서버 설정 기준 최대 15MB입니다.
