# Obsidian Kanban Board

A lightweight Kanban board for Obsidian using DataviewJS + a CSS snippet. No plugin install required beyond [Dataview](https://github.com/blacksmithgu/obsidian-dataview).

## Requirements

- Obsidian
- [Dataview](https://github.com/blacksmithgu/obsidian-dataview) plugin with **Enable JavaScript Queries** turned on

## Setup

1. Copy `kanban.css` to your vault's `.obsidian/snippets/` folder
2. Enable it in **Settings → Appearance → CSS Snippets**
3. Copy `templates/Kanban Board.md` and `templates/Kanban Task.md` to your vault's templates folder
4. Create a new note from the **Kanban Board** template
5. In edit mode, update the `FOLDER` variable to the relative path of your tasks folder

## Task Frontmatter

Each task is a markdown file with this frontmatter:

```yaml
---
title: My Task
status: to-do       # someday | to-do | in-progress | on-hold | done
priority: 0         # 0=none 1=lowest 2=low 3=medium 4=high 5=highest
order: 0
---
```

## Columns (customisable)

| id | Label |
|---|---|
| `someday` | Someday |
| `to-do` | To Do |
| `in-progress` | In Progress |
| `on-hold` | On Hold |
| `done` | Done |

Edit the `COLUMNS` array in the board template to add, remove, or rename columns.

## Features

- Drag and drop cards between columns
- Move cards up/down within a column
- Priority badges with colour coding
- State persisted automatically to frontmatter
- Full-width board layout
