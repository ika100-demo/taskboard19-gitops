---
spec_id: 001-persistent-tasks-in-postgres-and-task-ui
title: Persistent tasks in Postgres and task UI
status: approved
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres and task UI

## Problem

The Task API keeps tasks only in each running instance's memory: every restart or redeploy empties the list, and two instances would each show different tasks, so nobody can rely on a task still being there tomorrow. The web UI exists but shows only the template's landing page, so people can reach tasks only by calling the API by hand. This feature keeps tasks in a PostgreSQL database that belongs to the system, so they survive restarts and deploys, and lets people see and create tasks in the web UI.

## Stories

As an API client, I want tasks to survive restarts and redeploys of the Task API and to look the same from every instance, so that what I recorded is still there when I come back.
**Repos:** taskboard19-api, taskboard19-gitops

As a person using the taskboard, I want to open the web UI and see all tasks with their due dates, so that I know what needs doing without calling the API.
**Repos:** taskboard19-web, taskboard19-api, taskboard19-gitops

As a person using the taskboard, I want to create a task with a title and an optional due date in the web UI, so that I can record work without a technical tool.
**Repos:** taskboard19-web, taskboard19-api, taskboard19-gitops

## Acceptance criteria

- **AC-001.1** Given tasks "A" and "B" were created through `POST /tasks`, when every Task API instance is restarted or replaced by a new deploy and a client sends `GET /tasks`, then the response is 200 with both tasks in creation order, each with the same `id`, `title`, `due_date` and `created_at` returned when it was created, and the next task created gets an `id` greater than every existing one.
- **AC-001.2** Given the Task API runs as two or more instances, when a task is created through any one of them, then `GET /tasks` answered by any instance includes that task.
- **AC-001.3** Given the database is unreachable, when a client sends `POST /tasks` or `GET /tasks`, then the response is 503 with a JSON body holding `detail` "Task storage is unavailable", no task is created, and the Task API's readiness check answers 503; when the database is reachable again, the next request succeeds without restarting the Task API.
- **AC-001.4** Given tasks exist, when a person opens the web UI's home page, then it lists every task in creation order (oldest first), each showing its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none.
- **AC-001.5** Given no task exists, when a person opens the web UI's home page, then it shows "No tasks yet" and the create form.
- **AC-001.6** Given the home page is open, when a person enters the title "Buy milk", optionally picks a due date and submits the form, then the new task appears at the end of the list without the person reloading the page, the form is cleared, and `GET /tasks` on the Task API includes the task with that title and due date.
- **AC-001.7** Given the home page is open, when a person submits the form with a title that is empty, only spaces or longer than 200 characters, then the form shows "Title must be 1 to 200 characters and not only spaces." next to the title field, keeps the entered values, and no task is created.
- **AC-001.8** Given the Task API is unreachable or answers with a 5xx status, when a person opens the home page, then it shows "Tasks could not be loaded. Please try again later." instead of the list; and when a person submits the create form in that state, then it shows "The task could not be saved. Please try again." and keeps the entered values.

## Non-goals

- Editing, completing or deleting tasks, in the API or the UI.
- Filtering, sorting options, search and pagination of the list.
- Fields beyond title and due date.
- Login, user accounts and per-user task lists.
- Database backups, restore and staging/prod rollout; the database exists wherever the Task API runs (today dev only).
- Live updates of the list when someone else creates a task (a reload shows it).
- Carrying over tasks held in the in-memory Task API: they are lost when the database-backed version deploys, and the database starts empty.

## Assumptions

- **A1** Tasks are kept in one PostgreSQL database per environment that the system itself declares as a resource and runs in its own cluster, so it adds no paid service (other option: an external managed database; affects AC-001.1, AC-001.2, AC-001.3).
- **A2** The Task API's contract stays as it is: `POST /tasks` and `GET /tasks` with `id`, `title`, `due_date`, `created_at` and the existing validation; only where tasks are kept changes (other option: change the contract in the same spec; affects AC-001.1, AC-001.6).
- **A3** Ids stay positive integers assigned by the service and are never reused, but may have gaps (for example after a failed create) instead of always increasing by exactly 1 (other option: strictly gapless ids; affects AC-001.1).
- **A4** Database unavailability answers 503 with `detail` "Task storage is unavailable" and makes the Task API not ready, and it recovers by itself (other option: 500, or staying ready; affects AC-001.3).
- **A5** The task list and create form live on the web UI's home page and replace the template's landing content (other option: a separate tasks page; affects AC-001.4 to AC-001.8).
- **A6** The UI lists tasks oldest first, the order the API returns, and shows due dates as `YYYY-MM-DD` (other option: newest first, localized dates; affects AC-001.4, AC-001.6).
- **A7** The UI checks the title itself with the API's rules (1 to 200 characters, not only spaces) and uses a date picker for the due date, so the API's due-date format errors cannot occur from the UI (other option: show only the API's error messages; affects AC-001.7).
- **A8** UI wording: "No tasks yet", "No due date", "Title must be 1 to 200 characters and not only spaces.", "Tasks could not be loaded. Please try again later.", "The task could not be saved. Please try again." (other option: other wording; affects AC-001.4, AC-001.5, AC-001.7, AC-001.8).
- **A9** Anyone who can reach the web UI may see and create tasks without logging in, like the API today (decided 2026-11-15 in taskboard19-api spec 001: no authentication for now) (other option: require a login first; affects AC-001.4, AC-001.6).
- **A10** Tasks are kept indefinitely; there is no retention limit or cap on the number of tasks (other option: a cap or an expiry; affects AC-001.1, AC-001.4).

## Notes for the architect

- No database resource is declared in the system yet; the system needs a PostgreSQL resource and the Task API wired to it, and the web UI wired to the Task API inside each environment.
- This system spec changes taskboard19-api AC-001.7 (a restart empties the list and restarts ids at 1) and its A7; the Task API's component spec derived from the plan must supersede that criterion.

## Open questions

- ~~Tasks currently held in the running dev Task API's memory are lost when the database-backed version deploys; is that acceptable, with no carry-over? (suggested: yes, accept the loss; they vanish at every restart today anyway; affects AC-001.1)~~ Answered 2026-11-15: yes, accept the loss; nothing is carried over and the database starts empty. Folded into the non-goals.

## Changelog

- 2026-11-15 created
- 2026-11-15 amended: open question on in-memory tasks answered (accept the loss, the database starts empty), folded into the non-goals; no criteria changed
- 2026-11-15 approved
