# 📡 API 명세서


## 공통 규칙

**응답 형식** — JSON 응답은 모두 아래 형식입니다. 실패 시 `success: false`, `data: null` 이고 `message` 에 사유가 담깁니다.

```json
{ "success": true, "message": "성공", "data": { ... } }
```

본문이 JSON 이 아닌 예외:

- 예약 취소·영구 삭제 → `204 No Content` (본문 없음)
- Excel 다운로드·내보내기 → `.xlsx` 파일 (바이너리)

**권한** — 서버가 HTTP 메서드 기준으로 일괄 적용합니다. 표의 `Auth` 열은 아래 규칙의 결과입니다.

| 구분 | 필요한 권한 |
| ---- | ----------- |
| `/auth/**`, `/survey/**` | 없음 |
| `/admin/**` 의 GET (조회) | 직원  |
| `/admin/**` 의 POST · PUT · PATCH · DELETE | 관리자 |
| 예외 ① Excel 다운로드·내보내기 (GET) | 관리자 |

미인증은 `401`, 권한 부족은 `403` 으로 응답합니다.

---

### 인증 (공개)

| Method | Endpoint        | 설명                |
| ------ | --------------- | ------------------- |
| POST   | `/auth/login`   | 로그인 — access 는 바디, refresh 는 HttpOnly 쿠키(Path=/v1/auth). 실패 시 아이디 오류·비밀번호 오류 구분 없이 `401` |
| POST   | `/auth/reissue` | refresh 쿠키로 access + refresh 재발급 (rotation) |
| POST   | `/auth/logout`  | 이 기기 세션 삭제 + refresh 쿠키 만료 (항상 200) |

### 계정 관리

| Method | Endpoint                        | Auth       | 설명                 |
| ------ | ------------------------------- | ---------- | -------------------- |
| GET    | `/admin/accounts`               | 인증 필요  | 전체 계정 조회       |
| PATCH  | `/admin/accounts/{id}/role`     | 관리자 | 역할 변경  |
| DELETE | `/admin/accounts/{id}`          | 관리자 | 계정 삭제  |

### 예약 관리

| Method | Endpoint                              | Auth       | 설명                                             |
| ------ | ------------------------------------- | ---------- | ------------------------------------------------ |
| POST   | `/admin/reservations`                 | 관리자 | 예약 등록 (강의실·객실·식사 포함)                |
| GET    | `/admin/reservations?year=`           | 인증 필요  | 연도별 전체 예약 조회 (페이징 없음)              |
| GET    | `/admin/reservations/range`           | 인증 필요  | 날짜 범위 예약 조회 (`?from=&to=`, `yyyy-MM-dd`) |
| GET    | `/admin/reservations/search`          | 인증 필요  | 예약 검색 — 아래 파라미터 참고                   |
| GET    | `/admin/reservations/{id}`            | 인증 필요  | 예약 상세 조회                                   |
| PUT    | `/admin/reservations/{id}`            | 관리자 | 예약 전체 수정 (하위 데이터 재생성)              |
| DELETE | `/admin/reservations/{id}`            | 관리자 | 예약 취소 (상태를 "취소"로 변경, Soft) — `204`   |
| DELETE | `/admin/reservations/{id}/hard`       | 관리자 | 예약 영구 삭제 (하위 데이터 포함) — `204`        |
| GET    | `/admin/reservations/check-classroom` | 인증 필요  | 강의실 이용 가능 여부 확인 (`?classroom=&date=`) → `true`/`false` |

**`/admin/reservations/search` 파라미터** (모두 선택)

| 파라미터            | 설명                                                          |
| ------------------- | ------------------------------------------------------------- |
| `keyword`           | 단체명 · 담당자 · 연락처 · 예약코드                           |
| `status`            | 확정 / 예약 / 문의 / 취소                                     |
| `startDate`, `endDate` | 날짜 범위 (`yyyy-MM-dd`)                                   |
| `page`, `size`, `sort` | 페이징 — 기본 `size=20`, `createdAt` 내림차순              |

### Excel

| Method | Endpoint                                | Auth       | 설명                                          |
| ------ | --------------------------------------- | ---------- | --------------------------------------------- |
| GET    | `/admin/reservations/{id}/estimate`     | 관리자 | 견적서 Excel 다운로드                         |
| GET    | `/admin/reservations/{id}/trade`        | 관리자 | 거래명세서 Excel 다운로드                     |
| GET    | `/admin/reservations/{id}/confirmation` | 관리자 | 확인서 Excel 다운로드                         |
| GET    | `/admin/reservations/export`            | 관리자 | 전체 예약 데이터 내보내기                     |
| POST   | `/admin/reservations/import`            | 관리자 | 예약 데이터 Excel 가져오기 (`multipart/form-data`, 필드 `file`, 최대 10MB) |

### 설문

| Method | Endpoint                               | Auth       | 설명                                  |
| ------ | -------------------------------------- | ---------- | ------------------------------------- |
| POST   | `/admin/surveys/token/{reservationId}` | 관리자 | 설문 토큰(URL) 생성                   |
| GET    | `/admin/surveys/token/{reservationId}` | 인증 필요  | 예약별 설문 토큰 조회                 |
| GET    | `/admin/surveys/tokens`                | 인증 필요  | 전체 토큰 목록 조회                   |
| GET    | `/admin/surveys/{reservationId}`       | 인증 필요  | 예약별 설문 응답 조회                 |
| GET    | `/admin/surveys`                       | 인증 필요  | 전체 설문 응답 조회                   |
| GET    | `/admin/surveys/search`                | 인증 필요  | 설문 응답 검색·페이징 (`?keyword=`, 기본 `size=20`, `createdAt` 내림차순) |
| GET    | `/survey/check/{token}`                | 공개       | 토큰 유효성 확인                      |
| POST   | `/survey/{token}`                      | 공개       | 설문 응답 제출 (이미 제출했으면 `409`) |

### 대시보드

| Method | Endpoint           | Auth      | 설명                                                              |
| ------ | ------------------ | --------- | ----------------------------------------------------------------- |
| GET    | `/admin/dashboard` | 인증 필요 | 대시보드 전체 데이터 (KPI·오늘 강의실·월별 추이·만족도·최근 설문) |

### 앱 설정

| Method | Endpoint                          | Auth       | 설명                                  |
| ------ | --------------------------------- | ---------- | ------------------------------------- |
| GET    | `/admin/settings`                 | 인증 필요  | 설정 전체 조회 (KV Map)               |
| PUT    | `/admin/settings`                 | 관리자 | 설정 전체 저장                        |
| GET    | `/admin/settings/disabledClassroom` | 인증 필요 | 사용 중지 강의실 목록 조회            |
| PUT    | `/admin/settings/disabledClassroom` | 관리자 | 사용 중지 강의실 목록 저장 (문자열 배열) |
| GET    | `/admin/settings/disabledRoom`    | 인증 필요  | 사용 중지 객실 목록 조회              |
| PUT    | `/admin/settings/disabledRoom`    | 관리자 | 사용 중지 객실 목록 저장 (문자열 배열) |
