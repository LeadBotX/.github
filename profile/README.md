# LeadBotX

**AI-powered database management and natural language querying platform.**

LeadBotX lets teams connect their databases (PostgreSQL, MySQL) or upload data files (CSV, Excel), then explore, search, and query that data using plain English — powered by OpenAI. It includes a full role-based access control system, real-time team notifications, and a modern dashboard UI.

---

## What It Does

### Connect Any Data Source
Import a CSV/Excel file or plug in an existing PostgreSQL or MySQL database. LeadBotX introspects the schema, infers column types, and stores the metadata — ready for AI-powered querying in seconds.

### Ask Questions in Plain English
Type a question like *"Show me the top 10 customers by revenue"* or *"Find all orders placed in the last 30 days over $500"*. LeadBotX sends only the schema metadata (table names, column names, types) to OpenAI, which generates a safe query. The system validates it, executes it against the actual data source, and returns results — all without exposing raw data to the AI.

### Team Collaboration with RBAC
Invite team members by email, assign custom roles with table-level and column-level access restrictions. Members only see the data their role permits — enforced server-side on every query.

### Real-Time Notifications
Get instant WebSocket-based notifications when someone is invited to your database or accepts an invitation. All notifications appear in a live dropdown on the dashboard.

---

## Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        Frontend (React SPA)                      │
│  React 18 · TypeScript · Tailwind CSS 4 · Vite · Zustand        │
│  Firebase Auth (client) · Axios · Socket.IO Client               │
└──────────────────────────┬───────────────────────────────────────┘
                           │  REST API + WebSocket
                           ▼
┌──────────────────────────────────────────────────────────────────┐
│                      Backend (NestJS API)                         │
│  NestJS 11 · TypeScript · Mongoose 9 · Firebase Admin             │
│  OpenAI SDK · Multer · Cloudinary · Socket.IO                     │
├──────────────────────────────────────────────────────────────────┤
│                                                                   │
│  ┌─────────────┐  ┌──────────────┐  ┌──────────────────────────┐ │
│  │ Auth Module  │  │ User Module  │  │ Connection Module        │ │
│  │ Firebase     │  │ Profile CRUD │  │ Import CSV/Excel         │ │
│  │ Guard        │  │ Avatar       │  │ Connect PostgreSQL/MySQL │ │
│  └─────────────┘  └──────────────┘  │ Schema Introspection     │ │
│                                      │ SQL Client Pool          │ │
│  ┌──────────────────────────┐       └──────────────────────────┘ │
│  │ Dashboard Module         │                                     │
│  │ Connection list, summary │       ┌──────────────────────────┐ │
│  │ Table preview, pagination│       │ Role Management Module   │ │
│  └──────────────────────────┘       │ Role definitions (RBAC)  │ │
│                                      │ Member invite/remove     │ │
│  ┌──────────────────────────┐       │ Table + column rules     │ │
│  │ NL Query Module          │       └──────────────────────────┘ │
│  │ OpenAI integration       │                                     │
│  │ SQL safety validation    │       ┌──────────────────────────┐ │
│  │ MongoDB chunk executor   │       │ Notification Module      │ │
│  │ Saved/recent queries     │       │ CRUD + WebSocket push    │ │
│  │ Rate limiting (20/hr)    │       │ Real-time delivery       │ │
│  └──────────────────────────┘       └──────────────────────────┘ │
│                                                                   │
└──────────────────────────┬───────────────────────────────────────┘
                           │
              ┌────────────┼────────────────┐
              ▼            ▼                ▼
        ┌──────────┐ ┌──────────┐   ┌─────────────┐
        │ MongoDB  │ │ OpenAI   │   │ PostgreSQL  │
        │ Atlas    │ │ API      │   │ / MySQL     │
        │          │ │          │   │ (user's own │
        │ Users    │ │ Schema → │   │  database)  │
        │ Conns    │ │ SQL/Plan │   │             │
        │ Chunks   │ │          │   │ Query       │
        │ Roles    │ │ No raw   │   │ execution   │
        │ Notifs   │ │ data sent│   │             │
        └──────────┘ └──────────┘   └─────────────┘
```

---

## Full Tech Stack

### Frontend
| Technology | Purpose |
|-----------|---------|
| React 18 | UI framework |
| TypeScript 5.7 | Type safety |
| Vite 6.3 | Build tool and dev server |
| Tailwind CSS 4.1 | Utility-first styling |
| React Router 7.13 | Client-side routing |
| Zustand 5 | Lightweight state management |
| Axios 1.16 | HTTP client with auth interceptor |
| Firebase 12 (client) | Email/password authentication |
| Socket.IO Client 4 | Real-time notifications |
| Lucide React | Icon library |
| Web Speech API | Browser-native voice-to-text |

### Backend
| Technology | Purpose |
|-----------|---------|
| NestJS 11 | Modular server framework |
| TypeScript 5.7 | Type safety |
| Mongoose 9 | MongoDB ODM |
| Firebase Admin 13 | Server-side auth token verification |
| OpenAI SDK 6 | Natural language → SQL/plan conversion |
| pg 8 | PostgreSQL driver |
| mysql2 3 | MySQL driver |
| csv-parse 6 | CSV file parsing |
| xlsx 0.18 | Excel file parsing |
| Multer 2 | Multipart file upload handling |
| Cloudinary 2 | Cloud image storage (avatars) |
| Socket.IO 4 | WebSocket server for real-time push |
| Swagger 11 | Interactive API documentation |
| class-validator | Request DTO validation |

### Infrastructure
| Service | Purpose |
|---------|---------|
| MongoDB Atlas | Primary database (users, connections, chunks, roles, notifications) |
| Firebase | Authentication provider (email/password) |
| Cloudinary | Avatar image hosting |
| OpenAI API | AI model for natural language query translation |

---

## NLQ (Natural Language Query) — How It Works

LeadBotX translates plain English questions into executable queries using a two-step process:

### Step 1: Schema Context (sent to OpenAI)
Only metadata is sent — never raw data:
```
Table: customers (~5000 rows)
  name (string)
  email (string)
  revenue (number)
  created_at (date)
  Sample: {"name":"John","email":"john@example.com","revenue":1500}
```

### Step 2: AI Response (received from OpenAI)

**For SQL databases**, OpenAI returns a SQL query:
```json
{ "status": "ok", "sql": "SELECT name, revenue FROM customers ORDER BY revenue DESC LIMIT 10" }
```
This SQL is validated (SELECT-only, no forbidden patterns, table/column RBAC check, LIMIT cap at 500), then executed on the user's actual PostgreSQL or MySQL database.

**For CSV/Excel data**, OpenAI returns a structured plan:
```json
{
  "status": "ok",
  "plan": {
    "table": "orders",
    "filters": [{ "column": "total", "op": "gt", "value": 500 }],
    "sort": { "column": "total", "direction": "desc" },
    "limit": 50
  }
}
```
This plan is validated, then converted into a MongoDB aggregation pipeline (`$match → $unwind → $match → $sort → $limit → $project`) and executed against the stored data chunks.

### Security
- OpenAI never receives actual database rows
- All generated SQL is validated before execution
- Table and column access is enforced per user role
- Rate limited to 20 queries per user per hour
- MongoDB scan capped at 500,000 unwound rows

---

## Features at a Glance

| Feature | Description |
|---------|-------------|
| **Multi-source connections** | PostgreSQL, MySQL, CSV, Excel — all in one platform |
| **Natural language querying** | Ask questions in English, get table results |
| **Voice input** | Browser-side speech-to-text via Web Speech API |
| **SQL safety** | SELECT-only, pattern blocking, RBAC enforcement |
| **Role-based access (RBAC)** | Table-level and column-level restrictions per role |
| **Team collaboration** | Invite members by email, assign roles |
| **Real-time notifications** | WebSocket push for invites and team activity |
| **Data preview** | Browse tables with pagination and infinite scroll |
| **CSV export** | Export NLQ results as CSV files |
| **Query history** | Save up to 3 queries, track 3 recent per connection |
| **Schema refresh** | Re-introspect SQL databases on demand |
| **Connection management** | Rename, enable/disable, delete connections |
| **Profile management** | Name, email display, avatar upload via Cloudinary |
| **Swagger API docs** | Full interactive API documentation |
| **Responsive UI** | Mobile-friendly layout with Tailwind CSS |

---

## Repository Structure

```
LeadBotX/
├── .github/
│   └── profile/
│       └── README.md          ← You are here
│
├── backend/                   # NestJS API server
│   ├── src/                   # Source code (modules, services, controllers)
│   ├── .env.example           # Environment variable template
│   ├── package.json
│   └── README.md              # Backend-specific documentation
│
└── website/                   # React SPA frontend
    ├── src/                   # Source code (components, stores, lib)
    ├── .env.example           # Environment variable template
    ├── package.json
    └── README.md              # Frontend-specific documentation
```

---

## Quick Start

### Prerequisites
- Node.js 18+
- MongoDB Atlas cluster
- Firebase project (Authentication enabled)
- OpenAI API key
- Cloudinary account (for avatars)

### Backend
```bash
cd backend
cp .env.example .env           # Fill in real credentials
npm install
npm run dev                    # http://localhost:3000
```

### Frontend
```bash
cd website
cp .env.example .env           # Fill in Firebase config
npm install
npm run dev                    # http://localhost:5173
```

---

*Last updated: May 2026*
