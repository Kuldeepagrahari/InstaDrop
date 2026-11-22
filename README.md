# QuickDrop

**Share text, code, links, files and images across any device in seconds, using a simple room code.**

QuickDrop is a real-time sharing app built on the MERN stack. Create a room, share its code, and everyone who joins sees the same stream of messages and files instantly. Move a snippet from your laptop to your phone, or hand a file to a friend, without emailing yourself or signing into a messaging app.

[Live Demo](https://instadrop-sigma.vercel.app/) | [Report a Bug](../../issues)
<img width="1901" height="851" alt="image" src="https://github.com/user-attachments/assets/edd336ce-1605-48a6-bc08-86db18e8a70f" />

---

## Table of Contents

1. [Features](#features)
2. [How It Works](#how-it-works)
3. [Tech Stack](#tech-stack)
4. [Architecture](#architecture)
5. [Permission Model](#permission-model)
6. [Project Structure](#project-structure)
7. [Getting Started](#getting-started)
8. [Environment Variables](#environment-variables)
9. [API Reference](#api-reference)
10. [WebSocket Events](#websocket-events)
11. [Design Decisions](#design-decisions)
12. [Security](#security)
13. [Roadmap](#roadmap)
14. [Contributing](#contributing)
15. [License](#license)

---

## Features

- **Authentication and authorization.** Register and log in with secure sessions. Every protected action is authorized on the server.
- **Create rooms.** Create a room with any name. A unique, shareable room code is generated automatically.
- **Join by code.** Anyone with the code can join from any device or browser, including other people.
- **Send anything.** Post text, code snippets, links, files and pictures into a room.
- **Real-time sync.** Messages and uploads appear on every connected device instantly over WebSockets.
- **Copy and reuse.** One-click copy for text and code, and download for files, so content moves between all your devices.
- **Owner-only deletion.** Members can read and add content, but only the room owner can delete messages, files or the room itself.
- **Containerized.** The full stack runs with a single `docker compose up`.

---

## How It Works

1. **Sign up or log in.**
2. **Create a room** and give it a name. You become its **owner**.
3. **Share the room code** with your other devices or with other people.
4. **Join** from any device using the code.
5. **Drop** messages, files or images into the room. They sync to everyone in real time.
6. **Copy or download** whatever you need on the device you are using.
7. **Clean up.** The owner can delete any message or file, or delete the whole room.

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React.js |
| Backend | Node.js, Express.js |
| Database | MongoDB (Mongoose) |
| Real-time | WebSockets |
| Messaging | Apache Kafka |
| Cache and state | Redis |
| Auth | JWT |
| DevOps | Docker, Docker Compose |

---

## Architecture

```
 ┌──────────────┐        HTTPS (REST)        ┌─────────────────────┐
 │  React App   │ ─────────────────────────► │   Express API       │
 │  (any device)│                            │   (auth, rooms,     │
 │              │ ◄───── WebSocket ───────── │    uploads)         │
 └──────────────┘                            └──────────┬──────────┘
                                                        │
                          ┌─────────────────────────────┼───────────────────┐
                          ▼                             ▼                   ▼
                    ┌───────────┐               ┌──────────────┐     ┌────────────┐
                    │   Kafka   │ ────────────► │   Consumers  │ ──► │  MongoDB   │
                    │ (events)  │               │ (persist +   │     │ (users,    │
                    └───────────┘               │  broadcast)  │     │ rooms,     │
                                                └──────┬───────┘     │ messages)  │
                                                       ▼             └────────────┘
                                                 ┌───────────┐
                                                 │   Redis   │
                                                 │ (cache,   │
                                                 │  presence)│
                                                 └───────────┘
```

**Message flow**

1. A client sends a message or file over the WebSocket or REST API.
2. The server validates the JWT and checks the user's membership in the room.
3. The event is published to a Kafka topic.
4. A consumer persists it to MongoDB and broadcasts it to every client connected to that room.
5. Redis serves hot data, such as recent room messages and active room membership, so reads stay fast.

> Adjust this section to match exactly how your Kafka topics and Redis keys are used.

---

## Permission Model

| Action | Owner | Member (joined by code) |
|---|:---:|:---:|
| Join room with code | n/a | Yes |
| Read messages and files | Yes | Yes |
| Add message, file or picture | Yes | Yes |
| Copy or download content | Yes | Yes |
| Delete a message or file | Yes | No |
| Delete the room | Yes | No |

Ownership is checked on the server for every delete request. Hiding the delete button in the UI is not what enforces this rule.

---

## Project Structure

```
quickdrop/
├── client/                 # React frontend
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/          # WebSocket and auth hooks
│   │   └── api/
│   └── Dockerfile
├── server/                 # Express backend
│   ├── src/
│   │   ├── routes/
│   │   ├── controllers/
│   │   ├── middleware/     # auth, ownership checks, upload limits
│   │   ├── models/         # User, Room, Message
│   │   ├── kafka/          # producers and consumers
│   │   ├── redis/
│   │   └── websocket/
│   └── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## Getting Started

### Prerequisites

- Docker and Docker Compose
- (Optional, for running without Docker) Node.js 18+ and npm

### Run with Docker (recommended)

```bash
git clone https://github.com/<your-username>/quickdrop.git
cd quickdrop

cp server/.env.example server/.env
cp client/.env.example client/.env

docker compose up --build
```

Then open `http://localhost:3000`.

### Run locally without Docker

```bash
# Backend
cd server
npm install
npm run dev

# Frontend (new terminal)
cd client
npm install
npm start
```

You will need MongoDB, Redis and Kafka running locally and configured in your `.env` file.

---

## Environment Variables

**server/.env**

```env
PORT=5000
MONGO_URI=mongodb://mongo:27017/quickdrop
REDIS_URL=redis://redis:6379
KAFKA_BROKERS=kafka:9092
JWT_SECRET=change-me
JWT_EXPIRES_IN=7d
CLIENT_URL=http://localhost:3000
MAX_FILE_SIZE_MB=10
```

**client/.env**

```env
REACT_APP_API_URL=http://localhost:5000
REACT_APP_WS_URL=ws://localhost:5000
```

Never commit real secrets. Keep `.env` files in `.gitignore`.

---

## API Reference

> Update paths to match your routes.

### Auth

| Method | Endpoint | Description |
|---|---|---|
| POST | `/api/auth/register` | Create an account |
| POST | `/api/auth/login` | Log in and receive a JWT |

### Rooms

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| POST | `/api/rooms` | Yes | Create a room with a name |
| POST | `/api/rooms/join` | Yes | Join a room using its code |
| GET | `/api/rooms/:id` | Member | Get room details |
| DELETE | `/api/rooms/:id` | Owner | Delete the room |

### Messages and Files

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| GET | `/api/rooms/:id/messages` | Member | List room content |
| POST | `/api/rooms/:id/messages` | Member | Add text, code or link |
| POST | `/api/rooms/:id/files` | Member | Upload a file or image |
| DELETE | `/api/rooms/:id/messages/:msgId` | Owner | Delete a message or file |

---

## WebSocket Events

| Event | Direction | Purpose |
|---|---|---|
| `room:join` | Client to server | Subscribe to a room |
| `message:new` | Server to client | A message or file was added |
| `message:deleted` | Server to client | The owner removed an item |
| `room:deleted` | Server to client | The room was removed |
| `presence:update` | Server to client | Devices or users connected |

---

## Design Decisions

- **Why WebSockets?** Content must appear on all devices the moment it is dropped. Polling adds delay and wasted requests.
- **Why Kafka?** Publishing events to a queue decouples receiving a message from persisting and broadcasting it. A slow database write does not block the sender, and consumers can scale independently.
- **Why Redis?** Recent room content and active membership are read constantly. Caching them reduces load on MongoDB.
- **Why room codes instead of invites?** The goal is frictionless sharing across your own devices. A code works without needing the other device to be signed into the same account.
- **Why server-side ownership checks?** Authorization must never depend on the client. Every delete verifies the requester against the room's owner.
- **Why Docker?** A multi-service stack (API, client, MongoDB, Redis, Kafka) is painful to set up by hand. Compose makes it reproducible.

---

## Security

- Passwords are hashed before storage.
- JWT-based authentication on all protected REST routes and WebSocket connections.
- Server-side ownership and membership checks on every room action.
- File uploads are restricted by size and type.
- Rooms are accessible only by code, so use long, random codes and avoid sharing them publicly.

---

## Roadmap

- [ ] Room expiry and auto-delete (time-to-live)
- [ ] Optional room passwords
- [ ] End-to-end encryption for sensitive drops
- [ ] Drag-and-drop and clipboard paste uploads
- [ ] Mobile app (React Native)
- [ ] Rate limiting and abuse protection

---

## Contributing

Contributions are welcome.

1. Fork the repository
2. Create a branch: `git checkout -b feature/your-feature`
3. Commit your changes: `git commit -m "Add your feature"`
4. Push the branch: `git push origin feature/your-feature`
5. Open a pull request



Distributed under the MIT License. See `LICENSE` for details.

---

**Built by [Kuldeep Agrahari](https://github.com/Kuldeepagrahari)**
