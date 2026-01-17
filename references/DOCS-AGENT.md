# Docs Agent

**Role:** Documentation Turkey 🦃📝
**Phase:** Parallel with build, finalizes after features complete

## Purpose

Create comprehensive documentation that makes the project usable, maintainable, and understandable. Good docs mean fewer questions and faster onboarding.

## Responsibilities

1. **README.md** - Project overview, quick start
2. **CLAUDE.md** - AI assistant context file
3. **API Documentation** - Endpoint reference
4. **User Guide** - How to use the app
5. **Developer Guide** - How to contribute
6. **Architecture Docs** - System design decisions

## README.md Template

```markdown
# Project Name

One-line description of what this does.

## Features

- ✅ Feature one with brief description
- ✅ Feature two with brief description
- ✅ Feature three with brief description

## Quick Start

### Prerequisites

- Ruby 3.2+
- PostgreSQL 15+
- Node.js 20+ (for extension)

### Installation

```bash
# Clone the repository
git clone https://github.com/user/project.git
cd project

# Install dependencies
bundle install

# Setup database
rails db:create db:migrate db:seed

# Start the server
rails server
```

Open http://localhost:3000

### VS Code Extension

```bash
cd extension
npm install
npm run compile
# Press F5 to launch Extension Development Host
```

## Usage

### Dashboard
Navigate to the dashboard to see your focus sessions...

### Analytics
View your patterns at /analytics...

## Configuration

| Variable | Description | Default |
|----------|-------------|---------|
| `DATABASE_URL` | PostgreSQL connection string | localhost |
| `SESSION_GAP_MINUTES` | Minutes of inactivity before new session | 30 |

## API Reference

See [API.md](docs/API.md) for full documentation.

### Quick Reference

```bash
# Create heartbeat
curl -X POST http://localhost:3000/api/heartbeats \
  -H "Content-Type: application/json" \
  -d '{"project_name": "my-app", "branch": "main"}'

# Get current session
curl http://localhost:3000/api/current_session
```

## Development

```bash
# Run tests
bundle exec rspec

# Run linter
bundle exec rubocop

# Run with hot reload
bin/dev
```

## Deployment

See [DEPLOYMENT.md](docs/DEPLOYMENT.md)

## License

MIT

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)
```

## CLAUDE.md (Critical for AI Context)

```markdown
# CLAUDE.md

This file provides context for AI assistants working on this codebase.

## Project Overview

ADHD Hyperfocus Tracker - A passive focus tracking system with VS Code extension and Rails backend.

## Architecture

```
┌─────────────────┐     HTTP      ┌─────────────────┐
│  VS Code Ext    │──────────────▶│   Rails API     │
│  (TypeScript)   │   heartbeats  │   (Ruby)        │
└─────────────────┘               └────────┬────────┘
                                           │
                                           ▼
                                  ┌─────────────────┐
                                  │   PostgreSQL    │
                                  └─────────────────┘
```

## Key Concepts

### Passive Tracking
- Extension sends heartbeat every 2 minutes during coding activity
- No user interaction required
- Session boundaries detected by 30-minute gaps

### Session Detection
- Heartbeats within 30 minutes = same session
- Gap > 30 minutes = new session created
- Previous session auto-closed with end_time

## Directory Structure

```
/
├── api/                    # Rails backend
│   ├── app/
│   │   ├── controllers/
│   │   │   └── api/        # API endpoints
│   │   ├── models/         # Session, Heartbeat, User
│   │   └── services/       # SessionDetectionService
│   └── config/
│       └── routes.rb
│
├── src/                    # VS Code extension
│   ├── extension.ts        # Entry point
│   ├── heartbeatService.ts # Sends heartbeats
│   └── activityTracker.ts  # Detects coding activity
│
└── package.json            # Extension manifest
```

## Key Files

| File | Purpose |
|------|---------|
| `api/app/services/session_detection_service.rb` | Core session boundary logic |
| `api/app/controllers/api/heartbeats_controller.rb` | Receives heartbeats |
| `src/heartbeatService.ts` | Extension heartbeat sender |

## Data Models

### Session
```ruby
# Fields: project_name, branch, start_time, end_time, duration_minutes
# Relations: has_many :heartbeats
```

### Heartbeat
```ruby
# Fields: project_name, branch, file_type, timestamp
# Relations: belongs_to :session
```

## Common Tasks

### Add a new API endpoint
1. Add route in `config/routes.rb`
2. Create controller action
3. Add serializer if needed
4. Write request spec

### Modify session detection logic
1. Edit `SessionDetectionService`
2. Update specs in `spec/services/`
3. Consider impact on existing sessions

### Extension changes
1. Edit TypeScript in `src/`
2. Run `npm run compile`
3. Test with F5 in VS Code

## Testing

```bash
# All tests
bundle exec rspec

# Specific file
bundle exec rspec spec/services/session_detection_service_spec.rb

# Extension
cd extension && npm test
```

## Environment Variables

```bash
DATABASE_URL=postgresql://localhost/adhd_dev
SESSION_GAP_MINUTES=30  # Override default gap detection
```

## Gotchas

1. **Session boundary uses heartbeat timestamp, not updated_at**
   - The service checks last heartbeat time, not session record time
   
2. **Extension only sends heartbeats during activity**
   - No typing = no heartbeats
   - File switches count as activity

3. **API uses ActionController::Base, not ::API**
   - Hybrid app: JSON API + HTML dashboard
   - Don't change back to ::API

## Recent Decisions

- Chose 30-minute gap based on research on focus session patterns
- Using PostgreSQL for both dev and prod consistency
- Extension is purely passive - zero UI during focus
```

## API Documentation

```markdown
# API Reference

Base URL: `http://localhost:3000/api`

## Authentication

All endpoints require Bearer token authentication:
```
Authorization: Bearer <token>
```

## Endpoints

### Heartbeats

#### Create Heartbeat
`POST /heartbeats`

Records a heartbeat and joins/creates a session.

**Request:**
```json
{
  "heartbeat": {
    "project_name": "my-project",
    "branch": "feature/new-thing",
    "file_type": ".rb"
  }
}
```

**Response (201 Created):**
```json
{
  "session_id": 123,
  "heartbeat_id": 456,
  "session": {
    "id": 123,
    "project_name": "my-project",
    "branch": "feature/new-thing",
    "start_time": "2024-01-01T10:00:00Z",
    "duration_minutes": 45,
    "heartbeat_count": 23
  }
}
```

**Errors:**
- `401 Unauthorized` - Missing or invalid token
- `422 Unprocessable Entity` - Validation failed

---

### Sessions

#### Get Current Session
`GET /current_session`

Returns the active session (if any).

**Response (200 OK):**
```json
{
  "id": 123,
  "project_name": "my-project",
  "branch": "feature/new-thing",
  "start_time": "2024-01-01T10:00:00Z",
  "duration_minutes": 45,
  "heartbeat_count": 23
}
```

**Response (204 No Content):**
No active session.

---

#### List Sessions
`GET /sessions`

Returns paginated session history.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| page | integer | Page number (default: 1) |
| per_page | integer | Items per page (default: 20, max: 100) |

**Response (200 OK):**
```json
{
  "sessions": [...],
  "meta": {
    "current_page": 1,
    "total_pages": 5,
    "total_count": 98
  }
}
```
```

## User Guide Template

```markdown
# User Guide

## Getting Started

### What is ADHD Focus Tracker?

A passive tool that tracks your coding sessions without requiring any interaction. It runs in the background and helps you understand your focus patterns.

### How It Works

1. **Install the VS Code extension**
2. **Start coding** - that's it!
3. **Check your dashboard** when you want to see your patterns

The extension automatically detects when you're coding and sends small signals to the server. You don't need to start timers or click anything.

### Understanding Sessions

A "session" is a period of focused work. Sessions are automatically detected:

- **Session starts**: When you begin typing
- **Session continues**: As long as you're active within 30 minutes
- **Session ends**: After 30 minutes of inactivity

### Dashboard

The dashboard shows:
- **Today's Focus Time**: Total minutes coded today
- **Current Session**: What you're working on now
- **Recent Sessions**: Your latest focus sessions
- **Weekly Summary**: Patterns over the past week

### Analytics

View deeper insights:
- **Best Focus Hours**: When do you focus best?
- **Longest Sessions**: Your deepest work periods
- **Project Distribution**: Where does your time go?

## Tips for ADHD

1. **Don't check constantly** - Let it run in the background
2. **Review end of day** - Look at patterns, not real-time
3. **Celebrate wins** - Notice your long sessions!
4. **No guilt** - Short sessions are fine too
```

## Prompt Pattern

```
You are the Docs Agent creating project documentation.

Given the project, create:

1. **README.md**
   - Quick start (copy-paste-able)
   - Feature list
   - Basic usage
   - Links to detailed docs
   
2. **CLAUDE.md** (AI context file)
   - Architecture overview
   - Key concepts
   - Directory structure
   - Common tasks
   - Gotchas and decisions
   
3. **API.md**
   - All endpoints documented
   - Request/response examples
   - Error codes
   - Authentication
   
4. **User Guide**
   - Non-technical explanation
   - How to use each feature
   - Tips and best practices
   
5. **Developer Guide**
   - Setup instructions
   - Testing guide
   - Contribution process
   - Code standards

Documentation Standards:
- Copy-paste-able commands
- Real examples, not placeholders
- Keep it concise but complete
- Update when code changes

Output complete, markdown-formatted documentation files.
```

## Output Files

```
/
├── README.md               # Quick start, overview
├── CLAUDE.md              # AI assistant context
├── CONTRIBUTING.md        # How to contribute
└── docs/
    ├── API.md             # API reference
    ├── USER_GUIDE.md      # End-user documentation
    ├── DEVELOPER.md       # Developer setup
    ├── ARCHITECTURE.md    # System design
    └── DEPLOYMENT.md      # Deployment guide
```

## Success Criteria

Documentation complete when:
- [ ] README enables setup in < 5 minutes
- [ ] CLAUDE.md gives AI full context
- [ ] API docs cover all endpoints
- [ ] User can use app from guide alone
- [ ] Developer can contribute from docs alone
- [ ] No outdated information

## Anti-Patterns

**DON'T:**
- Write "TODO" in docs
- Use placeholder data
- Skip error documentation
- Leave commands untested
- Write walls of text

**DO:**
- Use real examples
- Test every command
- Keep it scannable
- Update with code changes
- Include troubleshooting
