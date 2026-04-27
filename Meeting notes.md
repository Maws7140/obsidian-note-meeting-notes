```dataviewjs
const DAYS_AHEAD = 14;
const SOURCES = ['"TaskNotes/Tasks"', '"School"'];
const PRIORITY_COLORS = { high: "pink", medium: "orange", low: "blue" };

const today = moment().startOf("day");
const end = moment(today).add(DAYS_AHEAD, "days").endOf("day");

function toMoment(v) {
  if (!v) return null;
  const m = moment(v.toJSDate ? v.toJSDate() : v);
  return m.isValid() ? m : null;
}

let items = [];
for (const src of SOURCES) {
  for (const p of dv.pages(src)) {
    const s = String(p.status ?? "").toLowerCase();
    if (s === "done" || s === "completed" || s === "cancelled") continue;
    const date = toMoment(p.due) || toMoment(p.scheduled) || toMoment(p.start);
    if (!date || !date.isSameOrBefore(end)) continue;
    const color = p.color ? String(p.color).replace(/"/g, "") : PRIORITY_COLORS[String(p.priority ?? "medium")] ?? "orange";
    const proj = p.projects ? String(p.projects) : p.class ? String(p.class).replace(/\[\[|\]\]/g, "") : "";
    items.push({ title: p.title ?? p.file.name, date, color, proj, path: p.file.path });
  }
}
items.sort((a, b) => a.date.valueOf() - b.date.valueOf());

const days = {};
for (const it of items) {
  const key = it.date.format("YYYY-MM-DD");
  if (!days[key]) days[key] = { date: it.date, items: [] };
  days[key].items.push(it);
}

const root = dv.el("div", "", { cls: "upcoming-events" });

root.innerHTML = `<div class="ue-header">
  <svg viewBox="0 0 24 24"><rect x="3" y="4" width="18" height="18" rx="2"></rect>
  <path d="M16 2v4"></path><path d="M8 2v4"></path><path d="M3 10h18"></path></svg>
  Upcoming events</div><div class="ue-body"></div>`;

const body = root.querySelector(".ue-body");
const dayKeys = Object.keys(days).sort();

if (!dayKeys.length) {
  body.innerHTML = '<div class="ue-empty">No upcoming events.</div>';
} else {
  for (const key of dayKeys) {
    const { date, items: dayItems } = days[key];
    const isToday = date.isSame(today, "day");
    const isOverdue = date.isBefore(today);
    const labelClass = isToday ? "ue-day-label-today" : isOverdue ? "ue-day-label-overdue" : "ue-day-label-future";
    const labelText = isToday ? "Today " + date.format("MMMM D") : isOverdue ? "Overdue " + date.format("MMMM D") : date.format("dddd MMMM D");

    const day = document.createElement("div");
    day.className = "ue-day";
    day.innerHTML = `<div class="ue-day-label ${labelClass}">${labelText}</div><div class="ue-day-events"></div>`;
    const eventsCol = day.querySelector(".ue-day-events");

    for (const it of dayItems) {
      const ev = document.createElement("div");
      ev.className = "ue-event";
      ev.dataset.color = it.color;
      ev.innerHTML = `<div class="ue-event-top"><div class="ue-dot"></div><div class="ue-event-title">${it.title}</div></div>` +
        (it.proj ? `<div class="ue-event-meta"><span class="ue-tag">${it.proj}</span></div>` : "");
      ev.onclick = () => app.workspace.openLinkText(it.path, "", false);
      eventsCol.appendChild(ev);
    }
    body.appendChild(day);
  }
}
```

---

```datacorejsx
function AudioManager() {
  const [recordings, setRecordings] = dc.useState([]);
  const [isRecording, setIsRecording] = dc.useState(false);
  const [searchQuery, setSearchQuery] = dc.useState('');
  const [showSearch, setShowSearch] = dc.useState(false);
  const [sortBy, setSortBy] = dc.useState('date-desc');
  const [showSort, setShowSort] = dc.useState(false);
  const [filterType, setFilterType] = dc.useState('all');
  const [showFilter, setShowFilter] = dc.useState(false);
  const [collapsed, setCollapsed] = dc.useState({});

  const fetchRecordings = () => {
    const exts = ['mp3', 'wav', 'm4a', 'webm', 'ogg'];
    setRecordings(app.vault.getFiles().filter(f => exts.includes(f.extension.toLowerCase())));
  };

  dc.useEffect(() => {
    fetchRecordings();
    const r1 = app.vault.on('create', fetchRecordings);
    const r2 = app.vault.on('delete', fetchRecordings);
    const r3 = app.vault.on('rename', fetchRecordings);
    return () => { app.vault.offref(r1); app.vault.offref(r2); app.vault.offref(r3); };
  }, []);

  const filtered = dc.useMemo(() => {
    let list = recordings;
    if (searchQuery) list = list.filter(f => f.name.toLowerCase().includes(searchQuery.toLowerCase()));
    if (filterType !== 'all') list = list.filter(f => f.extension.toLowerCase() === filterType);
    const sorted = [...list];
    switch (sortBy) {
      case 'date-desc': sorted.sort((a, b) => b.stat.ctime - a.stat.ctime); break;
      case 'date-asc': sorted.sort((a, b) => a.stat.ctime - b.stat.ctime); break;
      case 'name-asc': sorted.sort((a, b) => a.name.localeCompare(b.name)); break;
      case 'name-desc': sorted.sort((a, b) => b.name.localeCompare(a.name)); break;
      case 'size-desc': sorted.sort((a, b) => b.stat.size - a.stat.size); break;
      case 'size-asc': sorted.sort((a, b) => a.stat.size - b.stat.size); break;
    }
    return sorted;
  }, [recordings, searchQuery, sortBy, filterType]);

  // Group by month
  const months = dc.useMemo(() => {
    const groups = {};
    for (const f of filtered) {
      const d = new Date(f.stat.ctime);
      const key = d.getFullYear() + '-' + String(d.getMonth()).padStart(2, '0');
      const label = d.toLocaleDateString('en-US', { month: 'short', year: 'numeric' });
      if (!groups[key]) groups[key] = { label, files: [] };
      groups[key].files.push(f);
    }
    return Object.entries(groups).sort((a, b) => b[0].localeCompare(a[0]));
  }, [filtered]);

  const handleRecord = () => {
    if (isRecording) {
      app.commands.executeCommandById('audio-recorder:stop-recording');
      setIsRecording(false);
    } else {
      app.commands.executeCommandById('audio-recorder:start-recording');
      setIsRecording(true);
    }
  };

  const handlePlay = (file) => app.workspace.getLeaf(false).openFile(file);
  const handleDelete = async (file) => { if (confirm('Delete "' + file.name + '"?')) await app.vault.trash(file, true); };
  const handleRename = (e, file) => { e.stopPropagation(); app.fileManager.promptForFileRename(file); };

  const handleCreateNote = async (file) => {
    const now = new Date();
    const dateStr = now.toISOString().split('T')[0];
    const timeStr = now.toTimeString().split(' ')[0].replace(/:/g, '-');
    const title = 'Meeting ' + dateStr + ' ' + timeStr;
    const folder = 'Meeting notes';
    const path = folder + '/' + title + '.md';
    const content = '---\ndate: ' + dateStr + '\ntags: [meeting-note]\n---\n\n# ' + title + '\n\n![[' + file.name + ']]\n\n## Notes\n\n-\n';
    try {
      if (!await app.vault.adapter.exists(folder)) await app.vault.createFolder(folder);
      if (await app.vault.adapter.exists(path)) { new Notice('File already exists.'); return; }
      const f = await app.vault.create(path, content);
      new Notice('Created "' + title + '"');
      const leaf = app.workspace.getLeaf('tab');
      await leaf.openFile(f);
      app.workspace.setActiveLeaf(leaf, { focus: true });
    } catch (e) { new Notice('Error: ' + e.message); }
  };

  const toggleMonth = (key) => setCollapsed({ ...collapsed, [key]: !collapsed[key] });

  const showToolbar = showSearch || showSort || showFilter;

  return (
    <div className="audio-mgr">
      <div className="am-header">
        <div className="am-header-left">
          <svg viewBox="0 0 24 24"><path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2"></path><rect x="8" y="2" width="8" height="4" rx="1"></rect><path d="M9 14l2 2 4-4"></path></svg>
          Meeting notes
        </div>
        <div className="am-header-right">
          <button className={'am-icon-btn' + (showSort ? ' is-active' : '')} onClick={() => setShowSort(!showSort)} title="Sort">
            <svg viewBox="0 0 24 24"><path d="M11 5h10"></path><path d="M11 9h7"></path><path d="M11 13h4"></path><path d="M3 17l3 3 3-3"></path><path d="M6 18V4"></path></svg>
          </button>
          <button className={'am-icon-btn' + (showSearch ? ' is-active' : '')} onClick={() => setShowSearch(!showSearch)} title="Search">
            <svg viewBox="0 0 24 24"><circle cx="11" cy="11" r="8"></circle><path d="M21 21l-4.35-4.35"></path></svg>
          </button>
          <button className={'am-icon-btn' + (showFilter ? ' is-active' : '')} onClick={() => setShowFilter(!showFilter)} title="Filter">
            <svg viewBox="0 0 24 24"><path d="M4 6h16"></path><path d="M7 12h10"></path><path d="M10 18h4"></path></svg>
          </button>
          <button className={'am-new-btn' + (isRecording ? ' is-recording' : '')} onClick={handleRecord}>
            {isRecording ? 'Stop' : 'New'}
          </button>
        </div>
      </div>

      {showToolbar && (
        <div className="am-toolbar">
          {showSearch && <input className="am-search" type="text" placeholder="Search recordings..." value={searchQuery} onChange={(e) => setSearchQuery(e.target.value)} />}
          {showSort && (
            <select className="am-select" value={sortBy} onChange={(e) => setSortBy(e.target.value)}>
              <option value="date-desc">Newest</option>
              <option value="date-asc">Oldest</option>
              <option value="name-asc">Name A-Z</option>
              <option value="name-desc">Name Z-A</option>
              <option value="size-desc">Largest</option>
              <option value="size-asc">Smallest</option>
            </select>
          )}
          {showFilter && (
            <select className="am-select" value={filterType} onChange={(e) => setFilterType(e.target.value)}>
              <option value="all">All Types</option>
              <option value="mp3">MP3</option>
              <option value="wav">WAV</option>
              <option value="m4a">M4A</option>
              <option value="webm">WEBM</option>
              <option value="ogg">OGG</option>
            </select>
          )}
        </div>
      )}

      <div className="am-body">
        {months.length === 0 && (
          <div className="am-empty">{recordings.length === 0 ? 'No recordings found.' : 'No recordings match your filters.'}</div>
        )}
        {months.map(([key, group]) => {
          const isCollapsed = !!collapsed[key];
          return (
            <div className="am-month" key={key}>
              <div className="am-month-header" onClick={() => toggleMonth(key)}>
                <span className={'am-month-toggle' + (isCollapsed ? ' is-collapsed' : '')}>&#9660;</span>
                {group.label}
              </div>
              <div className={'am-month-items' + (isCollapsed ? ' is-collapsed' : '')}>
                {group.files.map(file => (
                  <RecordingRow key={file.path} file={file}
                    onPlay={handlePlay} onDelete={handleDelete}
                    onRename={handleRename} onCreateNote={handleCreateNote} />
                ))}
              </div>
            </div>
          );
        })}
      </div>
    </div>
  );
}

function RecordingRow({ file, onPlay, onDelete, onRename, onCreateNote }) {
  const [hasNote, setHasNote] = dc.useState(null);
  const [duration, setDuration] = dc.useState(null);

  dc.useEffect(() => {
    (async () => {
      for (const md of app.vault.getMarkdownFiles()) {
        try {
          const text = await app.vault.cachedRead(md);
          if (text.includes('![[' + file.name + ']]')) { setHasNote(md); return; }
        } catch (e) {}
      }
      setHasNote(false);
    })();
  }, [file.path]);

  dc.useEffect(() => {
    const audio = new Audio(app.vault.getResourcePath(file));
    audio.addEventListener('loadedmetadata', () => {
      const s = Math.round(audio.duration);
      if (!isFinite(s)) return;
      const h = Math.floor(s / 3600);
      const m = Math.floor((s % 3600) / 60);
      const sec = s % 60;
      setDuration(h > 0
        ? h + ':' + String(m).padStart(2, '0') + ':' + String(sec).padStart(2, '0')
        : m + ':' + String(sec).padStart(2, '0'));
    });
    return () => { audio.src = ''; };
  }, [file.path]);

  const handleNoteAction = async (e) => {
    e.stopPropagation();
    if (hasNote && hasNote !== false) {
      const leaf = app.workspace.getLeaf('tab');
      await leaf.openFile(hasNote);
      app.workspace.setActiveLeaf(leaf, { focus: true });
    } else {
      await onCreateNote(file);
    }
  };

  const d = new Date(file.stat.ctime);
  const shortDate = d.toLocaleDateString('en-US', { month: 'short', day: 'numeric' });
  const baseName = file.name.replace(/\.[^.]+$/, '');

  return (
    <div className="am-item" onClick={() => onPlay(file)}>
      <div className="am-item-icon">
        <svg viewBox="0 0 24 24"><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path><polyline points="14 2 14 8 20 8"></polyline><path d="M9 15l2 2 4-4"></path></svg>
      </div>
      <div className="am-item-info">
        <span className="am-item-title">{baseName}</span>
        {duration && <span className="am-item-date-inline">{duration}</span>}
      </div>
      <span className="am-item-date-short">{shortDate}</span>
      <div className="am-item-actions">
        <button className="am-icon-btn" onClick={handleNoteAction} title={hasNote ? 'Open note' : 'Create note'}>
          <svg viewBox="0 0 24 24">{hasNote && hasNote !== false
            ? <><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path><polyline points="14 2 14 8 20 8"></polyline><path d="M16 13H8"></path><path d="M16 17H8"></path><path d="M10 9H8"></path></>
            : <><path d="M14 2H6a2 2 0 0 0-2 2v16a2 2 0 0 0 2 2h12a2 2 0 0 0 2-2V8z"></path><polyline points="14 2 14 8 20 8"></polyline><path d="M12 18v-6"></path><path d="M9 15h6"></path></>
          }</svg>
        </button>
        <button className="am-icon-btn" onClick={(e) => onRename(e, file)} title="Rename">
          <svg viewBox="0 0 24 24"><path d="M17 3a2.85 2.83 0 1 1 4 4L7.5 20.5 2 22l1.5-5.5L17 3z"></path></svg>
        </button>
        <button className="am-icon-btn am-delete" onClick={(e) => { e.stopPropagation(); onDelete(file); }} title="Delete">
          <svg viewBox="0 0 24 24"><polyline points="3 6 5 6 21 6"></polyline><path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2"></path></svg>
        </button>
      </div>
    </div>
  );
}

return <AudioManager />;
```

