# TinyLink — A simple URL shortener

## Overview
TinyLink is a minimal URL shortener built with Next.js and Postgres (Neon). It supports:
- Create short links (optional custom code)
- Redirect (`/:code`) with 302
- Click counting and `last_clicked` timestamp
- Delete links
- Dashboard with list & controls
- Stats page `/code/:code`
- Health check `/healthz`
- API endpoints for autograding

## Tech
- Next.js
- node-postgres (`pg`)
- Plain CSS
- Deploy on Vercel + Neon

## Setup
1. Copy `.env.example` to `.env.local` and set `DATABASE_URL` and `NEXT_PUBLIC_BASE_URL`.
2. Run SQL in `schema.sql` to create `links` table.
3. `npm install`
4. `npm run dev` — open http://localhost:3000

## Endpoints (autograder)
- `GET /healthz` — 200 `{ ok: true, version: "1.0" }`
- `POST /api/links` — create link (409 if exists)
- `GET /api/links` — list links
- `GET /api/links/:code` — stats
- `DELETE /api/links/:code` — delete link
- `GET /:code` — redirect (302)
- `GET /code/:code` — UI stats page

## Deploy
See instructions in project root or deploy to Vercel and set environment variables.

## Tests
Use the curl commands listed in the repo to validate the autograder checks.

## Notes
- Codes must match `/^[A-Za-z0-9]{6,8}$/`.
