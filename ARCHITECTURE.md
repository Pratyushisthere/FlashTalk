## System Architecture Overview

This project is a full‑stack, real‑time chat application with a React client and a Node.js/Express backend. It uses JWT for stateless authentication, Socket.IO for realtime presence and messaging, MongoDB for persistence, and Cloudinary for image storage.

### High‑Level Diagram

Client (React + Vite)
  - AuthContext: session, JWT, axios config, Socket.IO client
  - ChatContext: users, messages, send/get, realtime subscriptions
  - UI: `Sidebar`, `ChatContainer`, `RightSidebar`, `LoginPage`, `ProfilePage`

Backend (Node.js + Express)
  - REST: auth and message routes
  - Socket.IO server: presence and realtime message delivery
  - Middleware: `protectRoute` (JWT auth)
  - Controllers: user and message operations
  - Persistence: MongoDB via Mongoose (`User`, `Message`)
  - Media: Cloudinary for image uploads

Infra/Config
  - `.env`: `MONGODB_URI`, `JWT_SECRET`, Cloudinary creds
  - Vite env: `VITE_BACKEND_URL`

---

## Client Architecture

- Framework/Build: React 18 + Vite, React Router, Tailwind‑style utility classes in CSS.
- State/Context:
  - `AuthContext` (`client/context/AuthContext.jsx`)
    - Persists JWT in `localStorage`, sets `axios.defaults.headers.token`.
    - `checkAuth` calls `GET /api/auth/check` to restore session and connects Socket.IO with `?userId`.
    - `login(state, credentials)` calls `POST /api/auth/{signup|login}`; stores token; connects socket; exposes `logout`, `updateProfile`.
    - Manages `onlineUsers` via `getOnlineUsers` Socket.IO broadcast.
  - `ChatContext` (`client/context/ChatContext.jsx`)
    - Fetches users: `GET /api/messages/users` (includes unseen counts per user).
    - Loads conversation: `GET /api/messages/:id` (also marks messages as seen on server).
    - Sends messages: `POST /api/messages/send/:id` with `{ text? , image? }`.
    - Subscribes to `newMessage` Socket.IO event; updates local `messages` or `unseenMessages` and calls `PUT /api/messages/mark/:id` for read receipts.

- Key UI Components:
  - `Sidebar`: user list, search, online/offline status, unseen counters, profile/logout.
  - `ChatContainer`: conversation view; sends text/images (FileReader to base64 for images).
  - `RightSidebar`: profile snippet for selected user and media gallery.

---

## Backend Architecture

- Entry: `server/server.js`
  - Express app + HTTP server for Socket.IO.
  - CORS enabled; JSON body limit 4MB (supports base64 images).
  - Routes: `/api/status`, `/api/auth/*`, `/api/messages/*`.
  - Socket.IO server with `userSocketMap` to track `userId -> socketId` and broadcast `getOnlineUsers` on connect/disconnect.
  - DB connection via `connectDB()` to `MONGODB_URI/chat-app`.

- Auth:
  - JWT creation: `lib/utils.js::generateToken(userId)`.
  - Middleware: `middleware/auth.js::protectRoute`
    - Reads token from `req.headers.token`.
    - `jwt.verify` with `JWT_SECRET` and attaches `req.user` (sans password).

- Data Models (Mongoose):
  - `User`: email, fullName, password (bcrypt hash), profilePic, bio, timestamps.
  - `Message`: senderId, receiverId, text, image, seen (default false), timestamps.

- Controllers:
  - `userController`
    - `signup`: validates input; hashes password; creates user; returns JWT.
    - `login`: verifies credentials; returns JWT and user.
    - `checkAuth`: returns `req.user` from middleware.
    - `updateProfile`: updates `bio`/`fullName`; optional Cloudinary upload for `profilePic`.
  - `messageController`
    - `getUsersForSidebar`: all users except requester; computes unseen counts per user.
    - `getMessages`: loads conversation between two users; marks incoming messages as seen.
    - `markMessageAsSeen`: sets a message `seen=true`.
    - `sendMessage`: persists message (optional Cloudinary upload for image) then emits `newMessage` to receiver’s socket if online.

- Realtime (Socket.IO):
  - Client connects with `?userId`. Server stores mapping and emits `getOnlineUsers` to all clients.
  - On `sendMessage`, server emits `newMessage` only to receiver’s socket if connected; client then acknowledges by calling `PUT /api/messages/mark/:id` when viewing.

- Media Storage:
  - Cloudinary configured in `lib/cloudinary.js` with environment variables.
  - Client sends base64 image; server uploads; stores `secure_url` in `Message.image`.

---

## Data Flow (Happy Paths)

1) Authentication
   - Sign up/Login → server returns `{ token, userData }` → client stores token → `axios.defaults.headers.token = token` → `checkAuth` confirms and `connectSocket` starts presence stream.

2) Presence
   - On Socket.IO connect/disconnect, server broadcasts `getOnlineUsers` with all online userIds → client updates presence badges.

3) Messaging
   - Select user → client `GET /api/messages/:id` → messages rendered; server `updateMany` marks as seen.
   - Send message → client `POST /api/messages/send/:id` → server saves and emits `newMessage` to receiver. If chat is open on receiver, client marks seen via `PUT /api/messages/mark/:id`; otherwise increments unseen counter.

4) Profile Update
   - Client submits form (with optional base64 image) → server updates user and optionally uploads to Cloudinary → returns updated user for immediate UI refresh.

---

## Security Considerations

- JWT stored in `localStorage` and set as a custom `token` header on requests.
- All protected routes guarded by `protectRoute` middleware with `jwt.verify`.
- Passwords hashed via `bcrypt` on signup.
- CORS enabled; Socket.IO uses `cors: { origin: "*" }` for simplicity (tighten for production).
- Image uploads: server‑side Cloudinary upload prevents direct client credentials exposure.

---

## Scaling and Improvements (Interview Talking Points)

- Authentication
  - Rotate/expire JWTs; add refresh tokens and short‑lived access tokens.
  - Move from custom `token` header to `Authorization: Bearer <token>` standard.

- Realtime
  - Horizontal scale Socket.IO with Redis adapter for pub/sub and shared `userSocketMap`.
  - Namespaced events and room joins per conversation for group chats.

- Data Model
  - Introduce `Conversation` collection to support group chats and message pagination.
  - Indexes on `{ senderId, receiverId, createdAt }` for efficient queries.

- Performance
  - Pagination/infinite scroll for messages; "load more" pattern.
  - Thumbnails and client‑side compression for images; signed Cloudinary transformations.

- Reliability
  - Acknowledge delivery/read receipts via Socket.IO acks; reconcile with REST marks.
  - Dead‑letter/queue for message persistence if DB or Cloudinary is transiently unavailable.

- Security/Privacy
  - Rate limiting, input validation (cele/zod), helmet; stricter CORS origins.
  - Encrypt JWT in HttpOnly cookies; CSRF protection for state‑changing routes if cookies used.

- Observability
  - Structured logging (pino/winston), request IDs, centralized logs.
  - Metrics (Prometheus) and tracing (OpenTelemetry) for API and Socket.IO events.

- Deployment
  - Containerize (Docker) with multi‑stage builds; environment‑based configs.
  - Use managed MongoDB (Atlas); set `MONGODB_URI` per environment.
  - Edge caching for static client via CDN; backend behind a reverse proxy.

---

## API Surface (Selected)

- Auth
  - `POST /api/auth/signup` → { fullName, email, password, bio }
  - `POST /api/auth/login` → { email, password }
  - `PUT /api/auth/update-profile` (auth)
  - `GET /api/auth/check` (auth)

- Messages
  - `GET /api/messages/users` (auth) → list users + unseen counts
  - `GET /api/messages/:id` (auth) → conversation and mark seen
  - `PUT /api/messages/mark/:id` (auth)
  - `POST /api/messages/send/:id` (auth) → text/image

---

## What to Highlight in an Interview

- Clear separation of concerns: REST for CRUD/persistence, Socket.IO for realtime delivery and presence.
- Stateless auth with middleware and context‑driven client integration.
- Thoughtful UX for presence, unseen counters, and media handling.
- Practical production hardening steps (Redis adapter, conversations, pagination, security headers, logging).

