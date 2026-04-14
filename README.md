# Kheliyega? - Real-Time Sports Matching & Commentary Backend

A learning project exploring modern Node.js best practices including real-time WebSocket communication, database migrations, security middleware, and API design.

> **Note:** This project was built for learning purposes, demonstrating practical patterns and architectural decisions in building a scalable sports backend.

## 📋 Overview

Kheliyega? is a backend service that manages sports matches and real-time commentary delivery. It demonstrates key concepts in:

- WebSocket server architecture for real-time data streaming
- Database schema design with Drizzle ORM
- Input validation with Zod schemas
- Security and rate limiting with Arcjet
- REST API design for sports data
- Real-time event broadcasting

## 🎯 Project Goals & Future Roadmap

### Core Features (Current)

- ✅ Match creation and listing (REST API)
- ✅ Live commentary streaming via WebSocket
- ✅ Real-time match event broadcasting
- ✅ Rate limiting and bot detection
- ✅ Database schema with migrations

### Upcoming (Frontend)

- 🚧 React/Next.js dashboard for match management
- 🚧 Live match viewer with real-time updates
- 🚧 Commentary UI for event streaming
- 🚧 WebSocket client integration
- 🚧 Authentication & user management

## 🏗️ Architecture

```
┌─────────────────────────────────────────┐
│           Express App                   │
│  (/matches, /matches/:id/commentary)    │
└──────────────┬──────────────────────────┘
               │
               ├──→ Arcjet (Rate Limiting, Bot Detection)
               │
               ├──→ Drizzle ORM + PostgreSQL
               │
┌──────────────▼──────────────────────────┐
│         HTTP Server                     │
│   (Enables both REST & WebSocket)       │
└──────────────┬──────────────────────────┘
               │
               └──→ WebSocket Server
                    (Real-time Broadcasting)
```

### Key Architectural Decision: Why Separate HTTP Server?

Express (`app`) alone cannot manage low-level TCP protocol switching. WebSockets require direct access to the underlying HTTP server to upgrade connections. Therefore:

```javascript
const server = http.createServer(app); // Low-level HTTP engine
attachWebSocketServer(server); // WebSocket upgrade point
```

Rather than using `app.listen()` (which hides the server), we explicitly manage the server to allow WebSocket integration. This is the standard pattern for mixed HTTP/WebSocket applications.

**Lesson Learned:** Node.js's http module is the foundation, Express is a layer on top. For advanced features like WebSockets, you need access to the underlying server.

## 📚 Key Lessons from This Project

### 1. **JavaScript Object References in Collections**

When using Maps/Sets with objects, remember you're storing references, not copies:

```javascript
const subscribers = matchSubscribers.get(matchId);
subscribers.delete(socket); // Modifies the original Set, not a copy
```

This is powerful for reactive updates but requires careful memory management.

### 2. **Handling Nullable Dates**

Directly passing `null` to `new Date()` produces unexpected behavior:

```javascript
// ❌ DON'T: new Date(null) → Current date/time (truthy!)
// ✅ DO: Check for null first, then parse only valid dates
if (endTime == null) {
  return MATCH_STATUS.LIVE;
}
const end = new Date(endTime);
if (Number.isNaN(end.getTime())) {
  return null; // Invalid date string
}
```

### 3. **Rate Limiting for WebSocket vs HTTP**

WebSocket connections are persistent and long-lived, unlike HTTP requests. Different rate limits are appropriate:

- HTTP: 50 requests per 10 seconds (standard API usage)
- WebSocket: 5 messages per 2 seconds (conversation-like cadence)

### 4. **Database Constraint Handling**

PostgreSQL unique constraint violations return error code `23505`. Distinguish between validation errors and business logic errors:

```javascript
if (error.cause?.code === "23505") {
  return res.status(409).json({ error: "Match already exists." });
}
```

### 5. **Input Validation Strategy**

Use Zod for both schema validation and runtime type safety. Combine field-level validation with cross-field refinements:

```javascript
createMatchSchema.superRefine((data, ctx) => {
  if (endTime <= startTime) {
    ctx.addIssue({ path: ["endTime"], message: "..." });
  }
});
```

## 🚀 Getting Started

### Prerequisites

- Node.js 18+
- PostgreSQL 14+
- npm

### Installation

```bash
# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Edit .env with your DATABASE_URL and ARCJET_KEY

# Run database migrations
npm run db:generate
npm run db:migrate
```

### Development

```bash
# Start server with auto-reload
npm run dev

# Server runs on http://localhost:8000
# WebSocket available at ws://localhost:8000/ws
```

### Production

```bash
npm start
```

## 📡 API Endpoints

### Matches

#### Get All Matches

```http
GET /matches?limit=50

Response: 200 OK
{
  "data": [
    {
      "id": 1,
      "sport": "football",
      "homeTeam": "Liverpool",
      "awayTeam": "Manchester United",
      "status": "live",
      "startTime": "2026-04-14T15:00:00Z",
      "endTime": "2026-04-14T17:00:00Z",
      "homeScore": 2,
      "awayScore": 1,
      "createdAt": "2026-04-14T14:50:00Z"
    }
  ]
}
```

#### Create Match

```http
POST /matches
Content-Type: application/json

{
  "sport": "football",
  "homeTeam": "Liverpool",
  "awayTeam": "Manchester United",
  "startTime": "2026-04-14T15:00:00Z",
  "endTime": "2026-04-14T17:00:00Z"
}

Response: 201 Created
```

### Commentary

#### Get Comments for Match

```http
GET /matches/:id/commentary?limit=100

Response: 200 OK
{
  "data": [
    {
      "id": 1,
      "matchId": 1,
      "minute": 45,
      "period": "first_half",
      "eventType": "goal",
      "actor": "Mohamed Salah",
      "team": "Liverpool",
      "message": "GOAL! Salah scores!",
      "tags": ["goal", "salah"],
      "createdAt": "2026-04-14T15:45:00Z"
    }
  ],
  "count": 1
}
```

#### Add Commentary

```http
POST /matches/:id/commentary
Content-Type: application/json

{
  "minute": 45,
  "sequence": 1,
  "period": "first_half",
  "eventType": "goal",
  "actor": "Mohamed Salah",
  "team": "Liverpool",
  "message": "GOAL! Salah scores!",
  "tags": ["goal", "salah"]
}

Response: 201 Created
```

### WebSocket

#### Subscribe to Match Updates

```javascript
const ws = new WebSocket("ws://localhost:8000/ws");

// Subscribe to match events
ws.send(
  JSON.stringify({
    action: "subscribe",
    matchId: 1,
  }),
);

// Listen for broadcast events
ws.onmessage = (event) => {
  const data = JSON.parse(event.data);
  console.log("Match update:", data);
  // Types: match_created, commentary_added
};
```

## 📊 Database Schema

### Matches Table

```sql
matches (
  id SERIAL PRIMARY KEY,
  sport TEXT NOT NULL,
  home_team TEXT NOT NULL,
  away_team TEXT NOT NULL,
  status match_status DEFAULT 'scheduled',
  start_time TIMESTAMP NOT NULL,
  end_time TIMESTAMP,
  home_score INTEGER DEFAULT 0,
  away_score INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(sport, home_team, away_team, start_time, end_time)
)
```

### Commentary Table

```sql
commentary (
  id SERIAL PRIMARY KEY,
  match_id INTEGER NOT NULL REFERENCES matches(id) ON DELETE CASCADE,
  minute INTEGER NOT NULL,
  sequence INTEGER NOT NULL,
  period TEXT NOT NULL,
  event_type TEXT NOT NULL,
  actor TEXT NOT NULL,
  team TEXT NOT NULL,
  message TEXT NOT NULL,
  metadata JSONB,
  tags JSONB,
  created_at TIMESTAMP DEFAULT NOW()
)
```

## 🛡️ Security Features

- **Rate Limiting:** Arcjet sliding window (HTTP: 50/10s, WebSocket: 5/2s)
- **Bot Detection:** Configurable bot detection rules
- **DDoS Shield:** Basic shield rules enabled
- **Input Validation:** Zod schemas for all user inputs
- **Error Handling:** Proper HTTP status codes and error messages
- **Database Constraints:** Unique indexes and referential integrity

## 📦 Dependencies

| Package        | Purpose                  |
| -------------- | ------------------------ |
| `express`      | HTTP server framework    |
| `ws`           | WebSocket implementation |
| `drizzle-orm`  | Type-safe ORM            |
| `pg`           | PostgreSQL driver        |
| `zod`          | Schema validation        |
| `@arcjet/node` | Rate limiting & security |
| `apminsight`   | Application monitoring   |
| `dotenv`       | Environment variables    |

## 🧪 Testing

WebSocket testing example (using wscat):

```bash
npm install wscat
npx wscat -c ws://localhost:8000/ws
# Interactive message sending/receiving
```

HTTP testing:

```bash
# Create a match
curl -X POST http://localhost:8000/matches \
  -H "Content-Type: application/json" \
  -d '{
    "sport": "football",
    "homeTeam": "Team A",
    "awayTeam": "Team B",
    "startTime": "2026-04-15T10:00:00Z",
    "endTime": "2026-04-15T12:00:00Z"
  }'

# List matches
curl http://localhost:8000/matches
```

## 📝 Development Notes

### Database Migrations

- Uses Drizzle migrations in `/drizzle` directory
- Run `npm run db:generate` after schema changes
- Run `npm run db:migrate` to apply pending migrations

### WebSocket Subscription Management

- Maintains a Map of match IDs to Sets of connected clients
- Per-socket subscription limit: 100 (prevents memory issues)
- Automatic cleanup when clients disconnect
- Validates match IDs to ensure positive integers ≤ 1,000,000

## 🎓 Learning Outcomes

This project demonstrates:

- ✅ Async/await patterns and error handling
- ✅ Real-time communication with WebSockets
- ✅ Database design and migrations
- ✅ REST API design principles
- ✅ Input validation and security
- ✅ Memory-efficient subscription management
- ✅ Proper middleware composition
- ✅ Environment configuration management

## 📄 License

ISC

## 👤 Author

Learning project for exploring Node.js best practices

---

**Version:** 1.0.0  
**Last Updated:** April 2026
