---
title: Kanban Board
---

```dataviewjs
// ── Config ────────────────────────────────────────────────────────────────────
const FOLDER = "+ Atlas - PROJECTS";
// Only files whose name starts with (PJ) appear as cards — no tag filter needed

// ── Statuses — sequential view columns + card left-border colour ──────────────
const STATUSES = [
  { id: "backlog",   label: "Backlog",    color: "#9E9E9E" },
  { id: "drafting",  label: "Drafting",   color: "#FFC107" },
  { id: "working",   label: "Working On", color: "#2196F3" },
  { id: "issue",     label: "Issue",      color: "#FF9800" },
  { id: "blocked",   label: "Blocked",    color: "#F44336" },
  { id: "done",      label: "Done",       color: "#4CAF50" },
  { id: "abandoned", label: "Abandoned",  color: "#424242" },
];

// ── Domains — domain view columns + tag filter chips ─────────────────────────
const DOMAINS = [
  { id: "p",      label: "Personal"      },
  { id: "biz",    label: "Business"      },
  { id: "boss",   label: "Boss"          },
  { id: "wop",    label: "Work Ops"      },
  { id: "wpjt",   label: "Work Project"  },
  { id: "career", label: "Career"        },
];

// ── Date filter options ───────────────────────────────────────────────────────
const CREATED_OPTIONS = [
  { id: "all", label: "All"  },
  { id: "1m",  label: "1M"   },
  { id: "3m",  label: "3M"   },
  { id: "6m",  label: "6M"   },
];
const DUE_OPTIONS = [
  { id: "all",           label: "All"      },
  { id: "overdue",       label: "Overdue"  },
  { id: "this-month",    label: "Month"    },
  { id: "this-quarter",  label: "Quarter"  },
  { id: "this-semester", label: "Semester" },
  { id: "this-year",     label: "Year"     },
];

// ── Tag aliases — map personal tags to domain ids ─────────────────────────────
const TAG_ALIASES = {
  "erik": "boss",
};

// ── Derived lookup ────────────────────────────────────────────────────────────
const STATUS_COLOR = Object.fromEntries(STATUSES.map(s => [s.id, s.color]));
const DOMAIN_IDS   = DOMAINS.map(d => d.id);

// ── Persistent state ───────────────────────────────────────────────────────────
const STATE_KEY = "kb_" + dv.currentFilePath.replace(/[^\w]/g, "_");
if (!window[STATE_KEY]) {
  window[STATE_KEY] = {
    ov: {}, orderOv: {},
    view: "sequential",
    tags: [],
    createdFilter: "all",
    dueFilter: "all",
  };
}
const STATE = window[STATE_KEY];

// ── Ephemeral drag state ───────────────────────────────────────────────────────
const DRAG = { path: null, overEl: null, overBefore: true };

// ── Load pages — only files starting with (PJ) appear as cards ───────────────
const pages = dv.pages(`"${FOLDER}"`)
  .where(p => p.file.name.startsWith("(PJ)"))
  .array();

// Purge stale overrides once dataview confirms the file was persisted
for (const p of Object.keys(STATE.ov)) {
  const pg = pages.find(x => x.file.path === p);
  if (pg && pg.status === STATE.ov[p]) delete STATE.ov[p];
}
for (const p of Object.keys(STATE.orderOv)) {
  const pg = pages.find(x => x.file.path === p);
  if (pg && pg.order === STATE.orderOv[p]) delete STATE.orderOv[p];
}

// ── File I/O — always background, never awaited in the hot path ───────────────
async function patchFrontmatter(path, patches) {
  const file = app.vault.getAbstractFileByPath(path);
  if (!file) return;
  let content = await app.vault.read(file);
  const m = content.match(/^---\r?\n([\s\S]*?)\r?\n---/);
  if (!m) return;
  let fm = m[1];
  for (const [key, val] of Object.entries(patches)) {
    const re = new RegExp(`^(${key}:\\s*).*$`, "m");
    fm = re.test(fm) ? fm.replace(re, `$1${val}`) : `${fm}\n${key}: ${val}`;
  }
  await app.vault.modify(file, content.replace(m[0], `---\n${fm}\n---`));
}

// ── Swap a domain tag in the tags YAML block ───────────────────────────────────
async function swapDomainTag(path, oldDomain, newDomain) {
  const file = app.vault.getAbstractFileByPath(path);
  if (!file) return;
  let content = await app.vault.read(file);
  const m = content.match(/^---\r?\n([\s\S]*?)\r?\n---/);
  if (!m) return;
  let fm = m[1];
  // Replace old domain tag line, or append new one if old not found
  const oldRe = new RegExp(`^(\\s*-\\s+)${oldDomain}\\s*$`, "m");
  if (oldRe.test(fm)) {
    fm = fm.replace(oldRe, `$1${newDomain}`);
  } else {
    // Add new domain under tags block
    fm = fm.replace(/(tags:[\s\S]*?)(\n\w)/, `$1\n  - ${newDomain}$2`);
  }
  await app.vault.modify(file, content.replace(m[0], `---\n${fm}\n---`));
}

// ── Task builder ───────────────────────────────────────────────────────────────
function buildTasks() {
  return pages.map((p, i) => {
    const allTags    = (p.file.tags || []).map(t => {
      const raw = String(t).replace(/^#/, "").toLowerCase();
      return TAG_ALIASES[raw] ?? raw;
    });
    let domainTags = allTags.filter(t => DOMAIN_IDS.includes(t));
    if (STATE.domainOv?.[p.file.path]) {
      const ov = STATE.domainOv[p.file.path];
      domainTags = [ov, ...domainTags.filter(t => t !== domainTags[0])];
    }
    return {
      path:     p.file.path,
      title:    p["project-title"] || p.file.name,
      status:   STATE.ov[p.file.path] ?? p.status ?? "backlog",
      priority:   p["project-priority"] === true,
      order:      STATE.orderOv[p.file.path] ?? (typeof p.order === "number" ? p.order : i),
      projectTag: p["project-tag"] ? String(p["project-tag"]).replace(/^#/, "") : null,
      domainTags,
      ctime:    p.file.ctime ?? null,
      due:      p.due        ?? null,
    };
  });
}

function byOrder(arr) {
  return [...arr].sort((a, b) => a.order - b.order);
}

// ── Combined filter ────────────────────────────────────────────────────────────
function applyFilters(tasks) {
  const now = dv.luxon.DateTime.now();

  return tasks.filter(task => {
    // Tag filter
    if (STATE.tags.length > 0 && !STATE.tags.some(tag => task.domainTags.includes(tag))) return false;

    // Created filter
    if (STATE.createdFilter !== "all") {
      const months = { "1m": 1, "3m": 3, "6m": 6 }[STATE.createdFilter];
      if (months && task.ctime) {
        if (task.ctime < now.minus({ months })) return false;
      }
    }

    // Due date filter
    if (STATE.dueFilter !== "all") {
      if (!task.due) return false; // no due date → exclude from any due filter
      const due = task.due;
      switch (STATE.dueFilter) {
        case "overdue":
          if (due >= now.startOf("day")) return false;
          break;
        case "this-month":
          if (due.year !== now.year || due.month !== now.month) return false;
          break;
        case "this-quarter":
          if (due.year !== now.year || due.quarter !== now.quarter) return false;
          break;
        case "this-semester": {
          const nowSem = now.month   <= 6 ? 1 : 2;
          const dueSem = due.month   <= 6 ? 1 : 2;
          if (due.year !== now.year || dueSem !== nowSem) return false;
          break;
        }
        case "this-year":
          if (due.year !== now.year) return false;
          break;
      }
    }

    return true;
  });
}

// ── Drop UI ────────────────────────────────────────────────────────────────────
function clearDropUI() {
  dv.container.querySelectorAll(".kb-col-over").forEach(el => el.classList.remove("kb-col-over"));
}
function clearCardDropUI() {
  dv.container.querySelectorAll(".kb-drop-above,.kb-drop-below")
    .forEach(el => { el.classList.remove("kb-drop-above"); el.classList.remove("kb-drop-below"); });
}

// ── Commit: drag to column background → land on top ───────────────────────────
function commitColumnChange(srcPath, targetColId) {
  const tasks = buildTasks();
  const src   = tasks.find(t => t.path === srcPath);
  if (!src) return;

  if (STATE.view === "sequential") {
    if (src.status === targetColId) return;
    const colTasks = byOrder(tasks.filter(t => t.status === targetColId));
    const newOrder = colTasks.length ? Math.min(...colTasks.map(t => t.order)) - 1 : 0;
    STATE.ov[srcPath] = targetColId;
    STATE.orderOv[srcPath] = newOrder;
    render();
    patchFrontmatter(srcPath, { status: targetColId, order: newOrder }).catch(console.error);

  } else {
    // Domain view: swap domain tag
    const oldDomain = src.domainTags[0];
    if (!oldDomain || oldDomain === targetColId) return;
    // Optimistic update in STATE via tag override
    STATE.domainOv = STATE.domainOv || {};
    STATE.domainOv[srcPath] = targetColId;
    render();
    swapDomainTag(srcPath, oldDomain, targetColId).catch(console.error);
  }
}

// ── Commit: drop onto a specific card (insert above or below) ─────────────────
function commitInsert(srcPath, targetPath, before) {
  const tasks  = buildTasks();
  const src    = tasks.find(t => t.path === srcPath);
  const target = tasks.find(t => t.path === targetPath);
  if (!src || !target || srcPath === targetPath) return;

  const colId  = STATE.view === "sequential" ? target.status : (target.domainTags[0] || "");
  const filtered = applyFilters(tasks);
  const colTasks = byOrder(
    STATE.view === "sequential"
      ? filtered.filter(t => t.status === colId)
      : filtered.filter(t => t.domainTags.includes(colId))
  );

  // Update status if cross-column
  const statusChanged = STATE.view === "sequential" && src.status !== colId;
  if (statusChanged) STATE.ov[srcPath] = colId;

  // Remove src, insert before/after target
  const rest      = colTasks.filter(t => t.path !== srcPath);
  const targetIdx = rest.findIndex(t => t.path === targetPath);
  const insertIdx = before ? targetIdx : targetIdx + 1;
  rest.splice(insertIdx, 0, src);

  rest.forEach((t, i) => { STATE.orderOv[t.path] = i; });
  render();
  rest.forEach((t, i) => { patchFrontmatter(t.path, { order: i }).catch(console.error); });
  if (statusChanged) patchFrontmatter(srcPath, { status: colId }).catch(console.error);
}

// ── Commit: move card up/down ──────────────────────────────────────────────────
function commitMove(srcPath, dir) {
  const tasks = buildTasks();
  const src = tasks.find(t => t.path === srcPath);
  if (!src) return;

  const filtered  = applyFilters(tasks);
  const groupKey  = STATE.view === "sequential" ? src.status : src.domainTags[0];
  if (!groupKey) return;

  const colTasks = byOrder(
    STATE.view === "sequential"
      ? filtered.filter(t => t.status === groupKey)
      : filtered.filter(t => t.domainTags.includes(groupKey))
  );

  const idx    = colTasks.findIndex(t => t.path === srcPath);
  const newIdx = dir === "up" ? idx - 1 : idx + 1;
  if (newIdx < 0 || newIdx >= colTasks.length) return;

  [colTasks[idx], colTasks[newIdx]] = [colTasks[newIdx], colTasks[idx]];
  colTasks.forEach((t, i) => { STATE.orderOv[t.path] = i; });
  render();
  colTasks.forEach((t, i) => { patchFrontmatter(t.path, { order: i }).catch(console.error); });
}

// ── Render a single card ───────────────────────────────────────────────────────
function renderCard(cardList, task, idx, colLength) {
  const now     = dv.luxon.DateTime.now();
  const isBlocked   = task.status === "blocked";
  const dueThisWeek = task.due && task.due >= now.startOf("day") && task.due <= now.endOf("week");
  const isUrgent    = !isBlocked && dueThisWeek && task.priority === true;

  let cls = "kb-card";
  if (isBlocked) cls += " kb-card-blocked";
  else if (isUrgent) cls += " kb-card-urgent";

  const card = cardList.createEl("div", { cls });
  card.draggable = true;
  card.style.borderLeftColor = STATUS_COLOR[task.status] || "#9E9E9E";

  card.addEventListener("dragstart", (e) => {
    DRAG.path = task.path;
    e.dataTransfer.effectAllowed = "move";
    e.dataTransfer.setData("text/plain", task.path);
    requestAnimationFrame(() => card.classList.add("kb-dragging"));
  });
  card.addEventListener("dragend", () => {
    card.classList.remove("kb-dragging");
    clearDropUI();
    clearCardDropUI();
    DRAG.path = null;
    DRAG.overEl = null;
  });

  // ── Card-level drop zone (insert above / below) ────────────────────────
  card.addEventListener("dragover", (e) => {
    if (!DRAG.path || DRAG.path === task.path) return;
    e.preventDefault();
    e.stopPropagation();
    e.dataTransfer.dropEffect = "move";
    const rect   = card.getBoundingClientRect();
    const before = e.clientY < rect.top + rect.height / 2;
    clearCardDropUI();
    card.classList.add(before ? "kb-drop-above" : "kb-drop-below");
    DRAG.overEl     = card;
    DRAG.overPath   = task.path;
    DRAG.overBefore = before;
  });
  card.addEventListener("dragleave", (e) => {
    if (!card.contains(e.relatedTarget)) {
      card.classList.remove("kb-drop-above", "kb-drop-below");
    }
  });
  card.addEventListener("drop", (e) => {
    e.preventDefault();
    e.stopPropagation();
    clearCardDropUI();
    clearDropUI();
    const srcPath = DRAG.path;
    DRAG.path = null;
    if (srcPath) commitInsert(srcPath, task.path, DRAG.overBefore);
  });

  const top = card.createEl("div", { cls: "kb-top" });

  if (task.priority === true) {
    const badge = top.createEl("span", { cls: "kb-badge" });
    badge.textContent = "⚡ Priority";
    badge.style.background = "#FF7881";
  }

  if (task.projectTag) {
    top.createEl("span", { cls: "kb-tag", text: `#${task.projectTag}` });
  }

  const moveWrap = top.createEl("div", { cls: "kb-move" });
  const btnUp    = moveWrap.createEl("button", { cls: "kb-move-btn", text: "↑" });
  btnUp.title    = "Move up";
  if (idx === 0) btnUp.disabled = true;
  btnUp.addEventListener("click", (e) => { e.stopPropagation(); commitMove(task.path, "up"); });

  const btnDown  = moveWrap.createEl("button", { cls: "kb-move-btn", text: "↓" });
  btnDown.title  = "Move down";
  if (idx === colLength - 1) btnDown.disabled = true;
  btnDown.addEventListener("click", (e) => { e.stopPropagation(); commitMove(task.path, "down"); });

  const a = card.createEl("a", { cls: "kb-title" });
  a.textContent = task.title;
  a.href = "#";
  a.onclick = (e) => {
    e.preventDefault();
    app.workspace.openLinkText(task.path, dv.currentFilePath, true);
  };

  // Due date display
  if (task.due) {
    const isOverdue = task.due < now.startOf("day");
    const dueEl = card.createEl("div", {
      cls: "kb-due" + (isOverdue ? " kb-due-overdue" : ""),
      text: (isOverdue ? "⚠ " : "📅 ") + task.due.toFormat("dd MMM yyyy"),
    });
  }
}

// ── Render ─────────────────────────────────────────────────────────────────────
function render() {
  const root    = dv.container;
  root.empty();

  const allTasks = buildTasks();
  const tasks    = applyFilters(allTasks);
  const columns  = STATE.view === "sequential" ? STATUSES : DOMAINS;

  // ── Toolbar ───────────────────────────────────────────────────────────────
  const toolbar = root.createEl("div", { cls: "kb-toolbar" });

  // Row 1: view + created + due
  const row1 = toolbar.createEl("div", { cls: "kb-toolbar-row" });

  // View switcher
  const viewSwitch = row1.createEl("div", { cls: "kb-view-switch" });
  ["sequential", "domain"].forEach(v => {
    const btn = viewSwitch.createEl("button", {
      cls: "kb-view-btn" + (STATE.view === v ? " kb-view-active" : ""),
      text: v === "sequential" ? "Sequential" : "Domain",
    });
    btn.addEventListener("click", () => { STATE.view = v; render(); });
  });

  row1.createEl("div", { cls: "kb-divider" });

  // Created filter
  const createdGroup = row1.createEl("div", { cls: "kb-filter-group" });
  createdGroup.createEl("span", { cls: "kb-filter-label", text: "Created:" });
  CREATED_OPTIONS.forEach(opt => {
    const btn = createdGroup.createEl("button", {
      cls: "kb-date-btn" + (STATE.createdFilter === opt.id ? " kb-date-active" : ""),
      text: opt.label,
    });
    btn.addEventListener("click", () => { STATE.createdFilter = opt.id; render(); });
  });

  row1.createEl("div", { cls: "kb-divider" });

  // Due filter
  const dueGroup = row1.createEl("div", { cls: "kb-filter-group" });
  dueGroup.createEl("span", { cls: "kb-filter-label", text: "Due:" });
  DUE_OPTIONS.forEach(opt => {
    const btn = dueGroup.createEl("button", {
      cls: "kb-date-btn" + (STATE.dueFilter === opt.id ? " kb-date-active" : ""),
      text: opt.label,
    });
    btn.addEventListener("click", () => { STATE.dueFilter = opt.id; render(); });
  });

  // Row 2: domain tag filter
  const row2 = toolbar.createEl("div", { cls: "kb-toolbar-row" });
  row2.createEl("span", { cls: "kb-filter-label", text: "Domain:" });
  const tagFilter = row2.createEl("div", { cls: "kb-tag-filter" });

  const btnAll = tagFilter.createEl("button", {
    cls: "kb-tag-btn" + (STATE.tags.length === 0 ? " kb-tag-active" : ""),
    text: "All",
  });
  btnAll.addEventListener("click", () => { STATE.tags = []; render(); });

  DOMAINS.forEach(domain => {
    const active = STATE.tags.includes(domain.id);
    const btn = tagFilter.createEl("button", {
      cls: "kb-tag-btn" + (active ? " kb-tag-active" : ""),
      text: `#${domain.id}`,
    });
    btn.title = domain.label;
    btn.addEventListener("click", () => {
      STATE.tags = active
        ? STATE.tags.filter(t => t !== domain.id)
        : [...STATE.tags, domain.id];
      render();
    });
  });

  // ── Board ─────────────────────────────────────────────────────────────────
  const board = root.createEl("div", { cls: "kb-board" });

  columns.forEach(col => {
    const colTasks = byOrder(
      STATE.view === "sequential"
        ? tasks.filter(t => t.status === col.id)
        : tasks.filter(t => t.domainTags.includes(col.id))
    );

    const colEl = board.createEl("div", { cls: "kb-col" });
    const hdr   = colEl.createEl("div", { cls: "kb-col-hdr" });

    if (STATE.view === "sequential" && col.color) {
      hdr.style.borderTopColor = col.color;
    }

    hdr.createEl("span", { text: col.label });
    hdr.createEl("span", { cls: "kb-count", text: String(colTasks.length) });

    const cardList = colEl.createEl("div", { cls: "kb-cards" });

    colEl.addEventListener("dragover", (e) => {
      if (!DRAG.path) return;
      e.preventDefault();
      e.dataTransfer.dropEffect = "move";
      colEl.classList.add("kb-col-over");
    });
    colEl.addEventListener("dragleave", (e) => {
      if (!colEl.contains(e.relatedTarget)) clearDropUI();
    });
    colEl.addEventListener("drop", (e) => {
      e.preventDefault();
      clearDropUI();
      const srcPath = DRAG.path;
      DRAG.path = null;
      if (srcPath) commitColumnChange(srcPath, col.id);
    });

    if (colTasks.length === 0) {
      cardList.createEl("div", { cls: "kb-empty", text: "Drop here" });
    }

    colTasks.forEach((task, idx) => renderCard(cardList, task, idx, colTasks.length));
  });
}

render();
```
