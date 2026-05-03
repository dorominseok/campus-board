## 3. 설계

### 3.1 데이터베이스 설계

MongoDB를 사용하며 Mongoose ODM으로 스키마를 정의한다.
주요 컬렉션은 User, Post, Notice, Comment, Like이며 ObjectId를 통해 컬렉션 간 참조 관계를 구성한다.
삭제 기능이 필요한 Post, Comment 컬렉션은 데이터 복구 가능성을 고려하여 Soft Delete 방식을 적용하였다.

#### ERD 다이어그램

<img width="1189" height="1175" alt="erd" src="https://github.com/user-attachments/assets/67e0dada-488f-419d-9b58-c1beca76c804" />

#### 컬렉션 스키마 정의

**User**

| 필드명 | 타입 | 설명 | 필수여부 |
|--------|------|------|----------|
| _id | ObjectId | MongoDB 기본 고유 식별자 (자동 생성) | 자동 |
| auth0_id | String | Auth0 고유 식별자 (sub) | 필수 |
| student_id | String | 학번 | 필수 |
| name | String | 이름 | 필수 |
| grade | Number | 학년 | 필수 |
| major | String | 전공 | 필수 |
| email | String | 이메일 | 필수 |
| role | String | 권한 (user / admin) | 기본값: user |
| created_at | Date | 생성일 (자동) | 자동 |

**Post**

| 필드명 | 타입 | 설명 | 필수여부 |
|--------|------|------|----------|
| _id | ObjectId | MongoDB 기본 고유 식별자 (자동 생성) | 자동 |
| board_id | ObjectId | 소속 게시판 (Board 참조) | 필수 |
| user_id | ObjectId | 작성자 (User 참조) | 필수 |
| title | String | 게시글 제목 | 필수 |
| content | String | 게시글 내용 | 필수 |
| view_count | Number | 조회수 | 기본값: 0 |
| like_count | Number | 좋아요 수 | 기본값: 0 |
| is_deleted | Boolean | 삭제 여부 (Soft Delete) | 기본값: false |
| created_at | Date | 생성일 (자동) | 자동 |
| updated_at | Date | 수정일 (자동) | 자동 |

**Notice**

| 필드명 | 타입 | 설명 | 필수여부 |
|--------|------|------|----------|
| _id | ObjectId | MongoDB 기본 고유 식별자 (자동 생성) | 자동 |
| list_no | Number | 공지 고유 번호 (크롤링 기준) | 필수 |
| title | String | 공지 제목 | 필수 |
| content | String | 공지 본문 | 선택 |
| author | String | 작성자 | 필수 |
| created_at | Date | 원본 공지 작성일 | 필수 |
| view_count | Number | 조회수 | 기본값: 0 |
| source_url | String | 원본 공지 URL | 필수 |
| is_pinned | Boolean | 공지 고정 여부 | 기본값: false |
| crawled_at | Date | 크롤링 시각 | 필수 |
| updated_at | Date | 갱신일 | 선택 |

**Comment**

| 필드명 | 타입 | 설명 | 필수여부 |
|--------|------|------|----------|
| _id | ObjectId | MongoDB 기본 고유 식별자 (자동 생성) | 자동 |
| post_id | ObjectId | 소속 게시글 (Post 참조) | 필수 |
| user_id | ObjectId | 작성자 (User 참조) | 필수 |
| content | String | 댓글 내용 | 필수 |
| is_deleted | Boolean | 삭제 여부 (Soft Delete) | 기본값: false |
| created_at | Date | 생성일 (자동) | 자동 |
| updated_at | Date | 수정일 (자동) | 자동 |

**Like**

| 필드명 | 타입 | 설명 | 필수여부 |
|--------|------|------|----------|
| _id | ObjectId | MongoDB 기본 고유 식별자 (자동 생성) | 자동 |
| post_id | ObjectId | 좋아요한 게시글 (Post 참조) | 필수 |
| user_id | ObjectId | 좋아요한 유저 (User 참조) | 필수 |
| created_at | Date | 생성일 (자동) | 자동 |

> Like 컬렉션은 (post_id + user_id) 복합 unique 인덱스 적용 — 중복 좋아요 방지

---

### 3.2 실시간 알림 API 설계

SSE(Server-Sent Events) 방식을 사용하여 서버에서 클라이언트로 단방향 실시간 알림을 전송한다.
WebSocket과 달리 별도 프로토콜 업그레이드 없이 HTTP 연결을 그대로 유지하며,
알림처럼 서버에서 클라이언트로 단방향 통신만 필요한 경우에 적합하다.
댓글 알림과 공지사항 알림 이벤트를 지원한다.

#### API 엔드포인트 명세

| 항목 | 내용 |
|------|------|
| 메서드 | GET |
| 경로 | /api/notifications/stream |
| 인증 | Auth0 JWT 토큰 필요 |
| Content-Type | text/event-stream |
| 설명 | SSE 방식으로 서버와 실시간 연결을 맺고 알림 이벤트를 수신 |

**이벤트 종류**

| 이벤트명 | 발생 조건 | 전송 데이터 |
|----------|-----------|-------------|
| connected | SSE 연결 성공 시 | `{ message }` |
| new_comment | 내 게시글에 댓글 작성 시 | `{ commentId, postId, content }` |
| new_notice | 새 공지사항 등록 시 (전체) | `{ noticeId, title, source }` |

#### 시스템 구조

실시간 알림은 sseManager 모듈을 중심으로 동작한다.
sseManager는 현재 연결된 클라이언트를 `Map<userId, res>` 형태로 관리하며,
알림 발생 시 해당 유저 또는 전체 유저에게 SSE 이벤트를 전송한다.

**모듈 간 연결 구조**

<img width="725" height="411" alt="module" src="https://github.com/user-attachments/assets/32820451-c094-48d9-9cc4-0a8a27916634" />

**클라이언트 연결 관리**

- 유저가 SSE 엔드포인트에 접속하면 userId를 키로 Map에 등록
- 유저가 연결을 끊으면 (`req.on('close')`) Map에서 자동 제거
- 동일 userId로 재접속 시 기존 연결을 덮어씀

---

### 3.3 순서 다이어그램

각 알림 기능의 처리 흐름을 나타낸다.
댓글 알림은 특정 유저에게만, 공지사항 알림은 접속 중인 전체 유저에게 전송되며
SSE 연결 흐름은 클라이언트가 스트림에 접속하여 이벤트를 수신하기까지의 과정을 나타낸다.

#### 댓글 알림 흐름

<img width="421" height="721" alt="notification1" src="https://github.com/user-attachments/assets/1e45c85e-1436-4b1d-833d-6402833c2117" />

#### 공지사항 알림 흐름

<img width="421" height="631" alt="notification2" src="https://github.com/user-attachments/assets/9bc16471-f328-4a6a-a418-6f6ee6e37a12" />

#### SSE 연결 흐름

![SSE 연결 순서도](sse_connection_flow.png)
