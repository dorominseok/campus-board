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

<img width="621" height="391" alt="제목 없는 다이어그램 drawio (10)" src="https://github.com/user-attachments/assets/892a2a47-f5d1-4b89-8b31-4e5abd724aaf" />

**클라이언트 연결 관리**

- 유저가 SSE 엔드포인트에 접속하면 userId를 키로 Map에 등록
- 유저가 연결을 끊으면 (`req.on('close')`) Map에서 자동 제거
- 동일 userId로 재접속 시 기존 연결을 덮어씀

---

### 3.3 순서 다이어그램

각 알림 기능의 처리 흐름을 나타낸다.
댓글 알림은 특정 유저에게만, 공지사항 알림은 접속 중인 전체 유저에게 전송되며
SSE 연결 흐름은 클라이언트가 스트림에 접속하여 이벤트를 수신하기까지의 과정을 나타낸다.

#### SSE 연결 흐름

<img width="426" height="891" alt="제목 없는 다이어그램 drawio (9)" src="https://github.com/user-attachments/assets/164cd271-7f36-4bb4-93fe-69f234fb81be" />


```
함수 handleSSEConnection(req, res):
  토큰 = req.headers.authorization
  만약 인증실패(토큰):
    반환 401 Unauthorized

  SSE 헤더 설정(res, "text/event-stream")
  userId = DB에서 사용자 조회(토큰.sub)
  addClient(userId, res)
  이벤트 전송(res, "connected")

  req.on("close"):
    removeClient(userId)
```


#### 댓글 알림 흐름

<img width="421" height="721" alt="notification1" src="https://github.com/user-attachments/assets/1e45c85e-1436-4b1d-833d-6402833c2117" />

```
함수 handleCommentCreate(req, res):
  토큰 = req.headers.authorization
  만약 인증실패(토큰):
    반환 401 Unauthorized

  댓글 = { post_id, user_id, content }
  DB에 댓글 저장(댓글)

  게시글 = DB에서 게시글 조회(post_id)
  만약 게시글.user_id != 댓글.user_id:
    notifyNewComment(게시글.user_id, 댓글)
```

#### 공지사항 알림 흐름

<img width="421" height="631" alt="notification2" src="https://github.com/user-attachments/assets/9bc16471-f328-4a6a-a418-6f6ee6e37a12" />

```
함수 crawlAndNotify():
  공지목록 = 학교사이트크롤링()

  반복 공지 in 공지목록:
    만약 DB에 공지 존재(공지.list_no):
      DB 업데이트(공지)
    아니면:
      DB에 공지 저장(공지)
      notifyNewNotice(공지)

함수 notifyNewNotice(공지):
  반복 (userId, res) in clients:
    이벤트 전송(res, "new_notice", 공지)
```

## 4. 구현

### 4.1 구현 환경

| 항목 | 내용 |
|------|------|
| 개발 언어 | JavaScript 24.11(Node.js) |
| 프레임워크 | Express.js 4.22 |
| 데이터베이스 | MongoDB Atlas |
| ODM | Mongoose 8.23 |
| 인증 | Auth0 (express-oauth2-jwt-bearer) |
| 개발 도구 | VS Code, Postman |
| 서버 구조 | REST API 서버 (클라이언트 요청 → Express 라우터 → MongoDB) |

### 4.2 구현 내용

#### 데이터베이스 스키마 연동

Mongoose ODM을 사용하여 MongoDB 컬렉션별 스키마를 정의한다.
각 모델은 필드 타입, 필수 여부, 기본값을 명시하며 컬렉션 간 참조는 ObjectId로 처리한다.

```javascript
// User 모델 
const userSchema = new mongoose.Schema({
  auth0_id: { type: String, required: true, unique: true },
  student_id: { type: String, required: true, unique: true },
  name: { type: String, required: true },
  grade: { type: Number, required: true },
  major: { type: String, required: true },
  email: { type: String, required: true, unique: true },
  role: { type: String, enum: ['user', 'admin'], default: 'user' }
}, { timestamps: { createdAt: 'created_at', updatedAt: false } });

#### SSE 인프라 구축

`sseManager.js`에서 연결된 클라이언트를 `Map<userId, res>` 형태로 관리한다.
클라이언트 등록/제거 및 이벤트 전송 기능을 별도 모듈로 분리하여 댓글 API와 크롤러에서 호출할 수 있도록 구현하였다.
```javascript
const clients = new Map();

function addClient(userId, res) { clients.set(userId, res); }
function removeClient(userId) { clients.delete(userId); }

function sendToUser(userId, event, data) {
  const res = clients.get(userId);
  if (res) {
    res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  }
}

function sendToAll(event, data) {
  clients.forEach((res) => {
    res.write(`event: ${event}\n`);
    res.write(`data: ${JSON.stringify(data)}\n\n`);
  });
}
```

#### SSE 스트림 엔드포인트

Auth0 인증을 거친 유저만 SSE 연결을 맺을 수 있도록 `authMiddleware`를 적용한다.
연결 성공 시 `connected` 이벤트를 전송하고, 연결 종료 시 Map에서 자동 제거된다.

```javascript
router.get('/stream', authMiddleware, async (req, res) => {
  const user = await User.findOne({ auth0_id: req.user.sub });
  if (!user) return res.status(401).json({ message: 'DB에 등록되지 않은 사용자입니다.' });

  res.setHeader('Content-Type', 'text/event-stream');
  res.setHeader('Cache-Control', 'no-cache');
  res.setHeader('Connection', 'keep-alive');
  res.flushHeaders();

  addClient(user._id.toString(), res);
  res.write(`event: connected\n`);
  res.write(`data: ${JSON.stringify({ message: '알림 연결 성공' })}\n\n`);

  req.on('close', () => { removeClient(user._id.toString()); });
});
```

#### 댓글 알림 구현

댓글 작성 시 게시글 작성자와 댓글 작성자가 다른 경우에만 알림을 전송한다.
본인 게시글에 본인이 댓글을 작성한 경우에는 알림을 전송하지 않는다.

```javascript
if (post.user_id.toString() !== user._id.toString()) {
  notifyNewComment(post.user_id, newComment);
}
```

#### 공지사항 알림 연결

크롤러에서 새 공지 저장 후 `notifyNewNotice()`를 호출하여 접속 중인 전체 유저에게 알림을 전송한다.

```javascript
await newNotice.save();
notifyNewNotice(newNotice);
```
