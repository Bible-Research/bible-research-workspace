# AI Agent Instructions - Bible Research Workspace

## Workspace Overview

This directory is a **monorepo workspace** containing two separate repositories for a Bible reading and study application:

| Repository | Role | Location |
|------------|------|----------|
| `bible_research` | Django REST API backend | `./bible_research/` |
| `reactive-bible` | React TypeScript frontend | `./reactive-bible/` |

The frontend (`reactive-bible`) communicates with the backend (`bible_research`) via REST API. They are developed and deployed independently.

---

## Repository Summaries

### `bible_research` — Backend

A **Django REST Framework** API that:
- Fetches Bible verse text and audio via the [Digital Bible Platform (DBT) API v4](https://www.faithcomesbyhearing.com/bible-brain/api-reference)
- Manages user annotations (notes and tags)
- Handles token-based authentication (DRF `TokenAuthentication`)
- Runs on PostgreSQL (production) / SQLite (development)

**Deployed**: Google App Engine (`https://bible-research-489314.ey.r.appspot.com`)

**Key docs**:
- `bible_research/README.md` — project overview
- `bible_research/DEVELOPER_GUIDE.md` — architecture, API endpoints, DB schema
- `bible_research/WINDSURF_AI_GUIDELINES.md` — detailed agent coding guidelines

### `reactive-bible` — Frontend

A **React 18 + TypeScript** SPA that:
- Displays Bible passages (KJV bundled locally, other translations via API)
- Provides audio playback (Howler.js + Media Session API)
- Manages notes and tags with a TipTap rich-text editor
- Uses Zustand for state management and Mantine v6 for UI
- Is built with Vite and deployed on Vercel

**Key docs**:
- `reactive-bible/README.md` — quick start and feature overview
- `reactive-bible/DEVELOPER_GUIDE.md` — full architecture and component guide
- `reactive-bible/CLAUDE.md` — concise agent-focused guide (start here for frontend)

---

## Critical Rules (apply to both repos)

1. **Python line length**: Max 79 characters per line (PEP 8 strict). No exceptions.
2. **No `bundle exec rspec`**: Not applicable here, but noted as a global user rule.
3. **No sensitive data in code**: API keys and secrets live in `bible_research/config.yaml` (gitignored).
4. **Commit message format**: `Type: Capitalized message` — e.g., `Feat: Add verse search endpoint`

---

## Where to Find Detailed Guidelines

For any work on either repository, **read the relevant guidelines file first**:

- **Backend agent guidelines**: `bible_research/WINDSURF_AI_GUIDELINES.md`
  - Code style, architecture patterns, DB conventions, security, testing, common tasks
- **Frontend agent guide**: `reactive-bible/CLAUDE.md`
  - Tech stack, project structure, state management, caching, API integration, testing patterns

---

## Quick Reference: Dev Commands

### Backend (`bible_research/`)
```bash
python manage.py runserver          # Start dev server (port 8000)
python manage.py makemigrations     # Create DB migrations
python manage.py migrate            # Apply migrations
python manage.py test annotations   # Run annotation tests
python manage.py test bible         # Run bible tests
```

### Frontend (`reactive-bible/`)
```bash
npm run dev       # Start dev server (http://localhost:5173)
npm run build     # TypeScript check + production build
npm run lint      # ESLint (zero warnings policy)
npm test          # Run tests
npm run coverage  # Coverage report
```
