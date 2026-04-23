<%*
const title = await tp.system.prompt("Project name");
const shortTitle = await tp.system.prompt("Kanban card title (~40 chars)", title);
const tag = await tp.system.prompt("Primary tag (e.g. #iris, #aperimail)");
const upLink = await tp.system.prompt("Parent MOC (e.g. ▼ MOC - IRIS)");
const domain = await tp.system.prompt("Domain tag (p/biz/boss/wop/wpjt/career)", "wpjt");
const priority = await tp.system.prompt("Priority? (true/false)", "false");
const due = await tp.system.prompt("Due date (YYYY-MM-DD, blank if none)", "");
await tp.file.rename("(PJ) " + title);
-%>
---
up: "[[<% upLink %>]]"
project-title: "<% shortTitle %>"
tags:
  - project
  - <% domain %>
  - <% tag.replace(/^#/, "") %>
status: backlog
priority: <% priority %>
date: <% tp.date.now("YYYY-MM-DD") %>
due: <% due %>
week: <% tp.date.now("YYYY-[W]WW") %>
modified: <% tp.date.now("YYYY-MM-DD") %>
project-active: true
project-status: "🔵 Planning"
project-next: ""
project-tag: "<% tag %>"
project-priority: <% priority %>
order: 0
---
# <% title %>

> [!important] Project Summary
> *One-paragraph description: what is this project, why does it matter, what does success look like?*

## Context & Objectives

**Objective:**
**Owner:** Patxi
**Key stakeholders:**
**Target completion:**

## Actions

*Tasks live here. Tag them with `<% tag %>` and add `📅 YYYY-MM-DD` for due dates so they surface in the Daily Note.*

- [ ] 📅 <% tp.date.now("YYYY-MM-DD") %> <% tag %>

## Decisions & Key Intel

*Capture decisions, deal intelligence, and strategic context as callouts:*

> [!note] Decision / Intel Title — YYYY-MM-DD
> *What was decided, why, what it means for next steps.*

## Dependencies & Waiting For

*Tag dependent tasks with `#waiting` so they surface in the Daily Note Waiting For section.*

- [ ] #waiting <% tag %>

## Risks & Blockers

> [!warning] Risk / Blocker Title
> *Description, impact, mitigation.*

## Links & Resources

-

## Status History

| Date | Status | Note |
|---|---|---|
| <% tp.date.now("YYYY-MM-DD") %> | 🔵 Planning | Project created |
