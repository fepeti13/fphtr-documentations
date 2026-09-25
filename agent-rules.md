# Agent rules

Rules for any local coding agent working in the fphtr repos. Read this before doing anything.

## 1. Plan before implementing

- Before writing a plan, **ask questions** about anything that is not yet decided or not clear — enough to be sure we share the same understanding. Ask only relevant questions, skip what's already answered by the conversation or the docs.
- Once questions are answered, write a **plan** (no code yet) and share it.
- Wait for explicit approval. Do not start implementing before the plan is approved.
- Implement only what the approved plan covers.

## 2. Keep it simple

- Always prefer the simplest solution that satisfies the approved plan.
- Do not add extra features, abstractions, config options, or "nice to haves" that weren't in the approved plan.
- If you notice a possible improvement or a risk, **flag it and describe it briefly** — do not implement it until explicitly told to.


## 3. Be straight to the point

- Clear, simple wording. No filler, no unrequested extras — in code, comments, docs, and replies alike.

## 4. Naming

- File names: kebab-case (`this-is-my-file-case`), unless a specific file or folder already has its own naming rule.

## 5. Credentials and secrets

- Never write real credentials anywhere (code, commit messages, docs, logs, config committed to the repo). Use placeholders.
- Some credentials are session-based and change every session (e.g. the university server's SSH password) — never hardcode or cache them across sessions.

## 6. Docs and system context

- The project's shared context, decisions and interface contracts live in the `fphtr-documentations` repo. Check it before assuming how another service in the system behaves.
- Keep code comments and doc updates minimal and accurate. Don't document features that don't exist yet.
- This repo is one part of a multi-repo system (UI, segmentation, recognition). Don't assume another service's behavior beyond what its interface contract documents.
