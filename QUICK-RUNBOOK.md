# Quick Runbook — Next.js Refactor Implementation

## Step 1 — Create the Implementation Plan

Start a new Claude Code session in this directory and say:

> use the writing-plans skill with docs/superpowers/specs/2026-03-20-nextjs-refactor-design.md

This reads the approved spec and produces a step-by-step implementation plan before any code is written.

---

## Step 2 — Execute the Plan

In a new session (or after Step 1 completes) say:

> use the executing-plans skill

This works through the plan with review checkpoints after each major step.

---

## Reference

- **Spec:** `docs/superpowers/specs/2026-03-20-nextjs-refactor-design.md`
- **Stack:** Next.js App Router · Prisma · PostgreSQL · Vercel
- **Local DB:** Docker Desktop required — `docker compose up -d` before `npm run dev`
