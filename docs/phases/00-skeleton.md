# Phase 0 — Skeleton

Started: 2026-11-08 · Finished: — · Tag: `phase-0`

## What I built

### Goal
A bare skeleton Rails app as a starting point of this learning journey.

### Decisions & why
- **Question 1:** Run `rails new --help` in a throwaway place such as /tmp, so it doesn't touch the repo. Find what `--api`, `-d postgresql` and `--skip-solid` each do.
  > - `--api` flag do create a new rails app but in API mode. so it only scaffolds the `backend` side, and didn't scaffold a fullstack rails app
  > - `-d postgresql` flag is an option to preconfigure which database would be used. so it could be mysql, trilogy, sqlite3, oracle, etc. but the flag clearly it preconfigures the database with `postgresql`
  > - `--skip-solid` means skipping the "Solid Trifecta" which refers to three official gems developed by Rails Core team: Solid Queue, Solid Cache, and Solid Cable (which all of those used a database preset by the Rails as a storage). usually this flag is applied because they rather use an external in-memory data store like Redis (for cache, and as the backing store for Sidekiq jobs).

- **Question 2:** Then answer in your own words: Rails defaults to SQLite, so why would a project that wants to practice scaling choose Postgres from the start? One sentence is enough.
  > ok so Rails defaulted to SQLite because its less complexity and performance wise on the local and production until it grows into a certain numbers where a transaction could have a heavy concurrent write loads. moreover SQLite doesn't scale horizontally, means only scale vertically on one single machine.

## What I broke

| Drill | Expected | Actually happened | Why |
|-------|----------|-------------------|-----|
|       |          |                   |     |

## Proof
- Specs / scripts and their results (benchmark numbers, query plans, etc.):

## Gate
- [ ] A fresh clone runs with one command and no manual steps
- [ ] Running the seeds twice produces identical data
- [ ] Every version is pinned (Ruby, gems, Docker images)
