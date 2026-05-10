[Русский](./decisions.ru.md) · **English**

# Architecture Notes — Content Generation Pipeline

> **Disclaimer.** This is a public architectural description of a real
> system the author worked on. Specific clients, domain names,
> financial indicators, source code, and proprietary implementation
> details are not disclosed. The content is limited to architectural
> decisions and principles publicly discussed for systems of this kind.

Walkthrough of key architectural decisions: context (what was being
solved) and implementation (how it was solved).

---

## 1 · Stages as reusable components

### Context

The system has several content-generation scenarios (long narrative
posts, short posts with media, series for specific campaigns). Some
steps repeat across scenarios, others differ.

### Implementation

A Stage is a module with explicit input/output contracts and a
handler. A Pipeline is an ordered list of references to stages with
their config, stored in the DB as a template. Data is passed between
stages via a serialized JSON-context that accumulates as the pipeline
progresses. A single stage implementation is used in multiple
scenarios; a new scenario is assembled by combining existing stages,
without code changes.

---

## 2 · In-UI pipeline builder

### Context

The target user is a product or content strategist who knows the
pipeline they need for a specific direction better than anyone, but
doesn't write code.

### Implementation

Stages are pulled from a registry maintained by the backend; the user
arranges them in the desired order via a visual builder; for each
stage there is a dynamic configuration form whose fields correspond to
its `config_schema`. The completed pipeline is saved to the DB as a
template and can be run against a specific brief.

---

## 3 · Human-in-the-loop as a first-class stage

### Context

LLMs at their current level don't give a stable quality guarantee:
they may produce generic, repetitive, or off-brand text. Most real
scenarios need a manual control point.

### Implementation

`manual-approval` / `manual-edit` are stage types just like any
automatic stage. On entering such a stage the pipeline transitions to
`awaiting_approval` and pauses; resumption only happens on an explicit
user action (approve / edit / reject).

The pipeline state machine has an `AwaitingApproval` state; a separate
approval service registers a pending action and notifies the user; the
UI assembles the approval queue; a user action updates
`stage_execution` and enqueues the next stage. Auto and manual stages
can be freely interleaved.

---

## 4 · Pipeline state in the main relational DB

### Context

Each pipeline run goes through several stages, each of which changes
status and accumulates context. This is a «second» kind of data
sitting alongside business data (templates, content, publications).

### Implementation

Templates, runs, and `stage_executions` live in the same PostgreSQL
as content and publications. Completion of a stage and committing its
result to business tables happen in a single DB transaction. One
observation surface — one DB, one admin tool, one backup.

---

## 5 · Storing credentials in the DB with transparent ORM-layer encryption

### Context

Users attach two kinds of credentials to projects: LLM provider API
keys and OAuth tokens for social platforms. The first is a billing
surface; the second is access to the customer's corporate accounts.

### Implementation

A custom column type at the ORM layer (`EncryptedText`) automatically
encrypts the value with Fernet on write and decrypts on read in the
Python process. The encryption key is stored in env, not in the DB.
In the UI, secret values are always masked; the full text is visible
only at creation/edit time. Backups and physical DB dumps don't
contain plaintext.

---

## 6 · Multi-channel publication schedule

### Context

A single content result is often published to several channels at
different times — e.g. a post on platform A at 10:00, its adapted
version on platform B at 14:00.

### Implementation

A scheduled publication is a DB record with links to the content
result and the platform integration, a `scheduled_at` field, and a
status machine (`scheduled` → `publishing` → `published` / `failed`).
A periodic task (Celery beat) on a small interval looks up
publications whose time has come, transitions them to `publishing`,
and hands them to a worker. Rescheduling means updating
`scheduled_at`. The UI always shows what will be published and when.
