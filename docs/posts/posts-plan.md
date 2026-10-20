# Milestone posts plan

One short post per milestone (300–600 words) on LinkedIn, Dev.to or Medium. Link the repo every time. Post as you learn, not after everything is finished.

## Post format that works

1. **Hook (1–2 lines):** the problem or surprise.
2. **Context:** what I was trying to do.
3. **What broke / what I learned:** the specific details.
4. **The fix or decision.**
5. **Link to the repo folder or file** and one screenshot.

## Planned posts

| # | Milestone | Working title | Draft status |
|---|---|---|---|
| 1 | Discovery done | "Before I touched Kubernetes: how I inventoried a production Docker Compose app" | ⬜ |
| 2 | Backups verified | "Backups aren't real until you've restored them: my MySQL + media restore test" | ⬜ |
| 3 | Pods running | "Moving Django, Celery and Redis to Kubernetes: what broke first" | ⬜ |
| 4 | State | "Docker volumes don't become PVCs: migrating MySQL to a StatefulSet" | ⬜ |
| 5 | Cutover | "Cutover day: re-pointing Nginx Proxy Manager to Kubernetes, with a rollback plan" | ⬜ |

## Draft starter for Post 1

> I've learned Kubernetes in theory. This week I started using it on a real project: a Django + Celery + MySQL + Redis app that runs on Docker Compose on a single VPS.
>
> Before writing a single manifest, I spent time on discovery: every container, port, volume and network, plus what I did *not* know yet. That list of unknowns turned out to be the most useful artifact.
>
> Three things I found: (1) the reverse proxy already owns ports 80/81/443, so Kubernetes can't just grab them; (2) Docker named volumes don't automatically become PersistentVolumeClaims; (3) Celery Beat must stay at a single replica or scheduled jobs run twice.
>
> My rule for the whole migration: inventory → backup → isolated test → validation → planned cutover → cleanup only after stability.
>
> I'm documenting everything, including mistakes, here: <repo link>

## Rules

- Never post secrets, real IPs you want private, or screenshots with tokens.
- Be honest about mistakes; that is what makes the posts credible.
- Reply to comments. Every conversation is networking.
