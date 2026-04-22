---
title: Kanban Board
---

```dataviewjs
// ── Config ────────────────────────────────────────────────────────────────────
const FOLDER = "UPDATE_THIS_TO_YOUR_TASKS_FOLDER_RELATIVE_PATH";

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

// ── Priority labels & colours ─────────────────────────────────────────────────
const P_LABEL = ["", "Lowest", "Low", "Medium", "High", "Highest"];
const P_COLOR = ["", "#89BDF4", "#89BDF4", "#FFE88B", "#FF7881", "#FF7881"];

// ── Derived lookup ────────────────────────────────────────────────────────────
const STATUS_COLOR = Object.fromEntries(STATUSES.map(s => [s.id, s.color]));
const DOMAIN_IDS   = DOMAINS.map(d => d.id);

// ── Persistent state ───────────────────────────────────────────────────────────
const STATE_KEY = "kb_" + dv.currentFilePath.replace(/[^\w]/g, "_");
if (!window[STATE_KEY]) {
  window[STATE_KEY] = { ov: {}, orderOv: {}, view: "sequential", tags: [] };
}
const STATE = window[STATE_KEY];

// ── Ephemeral drag state ───────────────────────────────────────────────────────
const DRAG = { path: null };

// ── Load pages ─────────────────────────────────────────────────────────────────
const pages = dv.pages(`"${FOLDER}"`).array();

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

// ── Task builder ───────────────────────────────────────────────────────────────
function buildTasks() {
  return pages.map((p, i) => {
    // Normalise tags: read from file.tags (picks up frontmatter + inline), strip #
    const allTags = (p.file.tags || []).map(t => String(t).replace(/^#/, "").toLowerCase());
    const domainTags = allTags.filter(t => DOMAIN_IDS.includes(t));
    return {
      path:     p.file.path,
      title:    p.title || p.file.name,
      status:   STATE.ov[p.file.path] ?? p.status ?? "backlog",
      priority: typeof p.priority === "number" ? p.priority : null,
      order:    STATE.orderOv[p.file.path] ?? (typeof p.order === "number" ? p.order : i),
      domainTags,
    };
  });
}

function byOrder(arr) {
  return [...arr].sort((a, b) => a.order - b.order);
}

// Filter tasks by selected domain tags (empty = show all)
function applyTagFilter(tasks) {
  if (!STATE.tags || STATE.tags.length === 0) return tasks;
  return tasks.filter(t => STATE.tags.some(tag => t.domainTags.includes(tag)));
}

// ── Drop UI ────────────────────────────────────────────────────────────────────
function clearDropUI() {
  dv.container.querySelectorAll(".kb-col-over").forEach(el => el.classList.remove("kb-col-over"));
}

// ── Commit: drag to column ─────────────────────────────────────────────────────
function commitColumnChange(srcPath, targetColId) {
  if (STATE.view !== "sequential") return; // domain view is read-only for drag
  const tasks = buildTasks();
  const src = tasks.find(t => t.path === srcPath);
  if (!src || src.status === targetColId) return;

  const colTasks = byOrder(tasks.filter(t => t.status === targetColId));
  const newOrder = colTasks.length ? Math.max(...colTasks.map(t => t.order)) + 1 : 0;

  STATE.ov[srcPath] = targetColId;
  STATE.orderOv[srcPath] = newOrder;
  render();
  patchFrontmatter(srcPath, { status: targetColId, order: newOrder }).catch(console.error);
}

// ── Commit: move card up/down within column ────────────────────────────────────
function commitMove(srcPath, dir) {
  const tasks = buildTasks();
  const src = tasks.find(t => t.path === srcPath);
  if (!src) return;

  const filtered = applyTagFilter(tasks);
  const groupKey = STATE.view === "sequential" ? src.status : src.domainTags[0];
  if (!groupKey) return;

  const colTasks = byOrder(
    STATE.view === "sequential"
      ? filtered.filter(t => t.status === groupKey)
      : filtered.filter(t => t.domainTags.includes(groupKey))
  );

  const idx = colTasks.findIndex(t => t.path === srcPath);
  const newIdx = dir === "up" ? idx - 1 : idx + 1;
  if (newIdx < 0 || newIdx >= colTasks.length) return;

  [colTasks[idx], colTasks[newIdx]] = [colTasks[newIdx], colTasks[idx]];
  colTasks.forEach((t, i) => { STATE.orderOv[t.path] = i; });
  render();
  colTasks.forEach((t, i) => { patchFrontmatter(t.path, { order: i }).catch(console.error); });
}

// ── Render a single card ───────────────────────────────────────────────────────
function renderCard(cardList, task, idx, colLength) {
  const card = cardList.createEl("div", { cls: "kb-card" });
  card.draggable = STATE.view === "sequential";
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
    DRAG.path = null;
  });

  const top = card.createEl("div", { cls: "kb-top" });

  if (task.priority !== null) {
    const badge = top.createEl("span", { cls: "kb-badge" });
    badge.textContent = P_LABEL[task.priority] ?? `P${task.priority}`;
    badge.style.background = P_COLOR[task.priority] ?? "#707BC2";
  }

  if (task.domainTags.length > 0) {
    const tagWrap = top.createEl("div", { cls: "kb-tags" });
    task.domainTags.forEach(tag => tagWrap.createEl("span", { cls: "kb-tag", text: `#${tag}` }));
  }

  const moveWrap = top.createEl("div", { cls: "kb-move" });

  const btnUp = moveWrap.createEl("button", { cls: "kb-move-btn", text: "↑" });
  btnUp.title = "Move up";
  if (idx === 0) btnUp.disabled = true;
  btnUp.addEventListener("click", (e) => { e.stopPropagation(); commitMove(task.path, "up"); });

  const btnDown = moveWrap.createEl("button", { cls: "kb-move-btn", text: "↓" });
  btnDown.title = "Move down";
  if (idx === colLength - 1) btnDown.disabled = true;
  btnDown.addEventListener("click", (e) => { e.stopPropagation(); commitMove(task.path, "down"); });

  const a = card.createEl("a", { cls: "kb-title" });
  a.textContent = task.title;
  a.href = "#";
  a.onclick = (e) => {
    e.preventDefault();
    app.workspace.openLinkText(task.path, dv.currentFilePath, true);
  };
}

// ── Render ─────────────────────────────────────────────────────────────────────
function render() {
  const root = dv.container;
  root.empty();

  const allTasks  = buildTasks();
  const tasks     = applyTagFilter(allTasks);
  const columns   = STATE.view === "sequential" ? STATUSES : DOMAINS;

  // ── Toolbar ──────────────────────────────────────────────────────────────
  const toolbar = root.createEl("div", { cls: "kb-toolbar" });

  // View switcher
  const viewSwitch = toolbar.createEl("div", { cls: "kb-view-switch" });
  ["sequential", "domain"].forEach(v => {
    const btn = viewSwitch.createEl("button", {
      cls: "kb-view-btn" + (STATE.view === v ? " kb-view-active" : ""),
      text: v.charAt(0).toUpperCase() + v.slice(1),
    });
    btn.addEventListener("click", () => { STATE.view = v; render(); });
  });

  // Tag filter chips
  const tagFilter = toolbar.createEl("div", { cls: "kb-tag-filter" });
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

    // Sequential view: colour the column header top border by status
    if (STATE.view === "sequential" && col.color) {
      hdr.style.borderTopColor = col.color;
    }

    hdr.createEl("span", { text: col.label });
    hdr.createEl("span", { cls: "kb-count", text: String(colTasks.length) });

    const cardList = colEl.createEl("div", { cls: "kb-cards" });

    colEl.addEventListener("dragover", (e) => {
      if (!DRAG.path || STATE.view !== "sequential") return;
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
