<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Public Directory</title>
  <style>
    :root{--bg:#0b0c10;--panel:#111319;--muted:#9aa0a6;--text:#e6e6e6;--accent:#4dabf7;--chip:#1b1f2a;--border:#2a2f3a}
    html,body{height:100%}
    body{margin:0;background:var(--bg);color:var(--text);font:14px/1.45 system-ui,-apple-system,Segoe UI,Roboto,Ubuntu,Cantarell,Noto Sans,Helvetica,Arial}
    .wrap{max-width:1100px;margin:auto;padding:24px}
    .card{background:var(--panel);border:1px solid var(--border);border-radius:16px;box-shadow:0 1px 0 rgba(0,0,0,.3);}
    .header{display:flex;flex-wrap:wrap;gap:12px;align-items:center;justify-content:space-between;padding:16px 18px;border-bottom:1px solid var(--border)}
    .controls{display:flex;flex-wrap:wrap;gap:12px;align-items:center}
    input[type="search"], select{background:#0e1116;color:var(--text);border:1px solid var(--border);border-radius:10px;padding:10px 12px}
    .chips{display:flex;flex-wrap:wrap;gap:8px}
    .chip{background:var(--chip);border:1px solid var(--border);border-radius:999px;padding:6px 10px;cursor:pointer;user-select:none}
    .chip.active{outline:2px solid var(--accent);}
    table{width:100%;border-collapse:collapse}
    th,td{padding:12px 14px;border-bottom:1px solid var(--border);vertical-align:top}
    th{color:var(--muted);text-align:left;font-weight:600}
    .muted{color:var(--muted)}
    .footer{display:flex;gap:8px;align-items:center;justify-content:space-between;padding:12px 16px}
    .pager{display:flex;gap:6px;flex-wrap:wrap}
    .btn{background:#0e1116;border:1px solid var(--border);color:var(--text);border-radius:8px;padding:8px 10px;cursor:pointer}
    .btn[disabled]{opacity:.5;cursor:not-allowed}
    .count{color:var(--muted)}
    .hl{background:rgba(77,171,247,.2)}
    .tag{display:inline-block;background:#12202d;border:1px solid #244a6a;border-radius:6px;padding:2px 6px;margin-right:6px}
    a{color:var(--accent)}
    @media (max-width:700px){
      .tablewrap{overflow:auto}
      th:nth-child(3), td:nth-child(3){display:none}
    }
  </style>
</head>
<body>
  <div class="wrap">
    <div class="card">
      <div class="header">
        <div class="controls">
          <input id="q" type="search" placeholder="Search name, description, tags…" aria-label="Search" />
          <select id="group" aria-label="Filter by group">
            <option value="">All groups</option>
          </select>
          <select id="pageSize" aria-label="Items per page">
            <option>10</option>
            <option selected>25</option>
            <option>50</option>
            <option>100</option>
          </select>
        </div>
        <div class="chips" id="facets" aria-label="Quick filters"></div>
      </div>

      <div class="tablewrap">
        <table>
          <thead>
            <tr>
              <th style="width:22%">Name</th>
              <th style="width:48%">Description</th>
              <th style="width:15%">Group</th>
              <th style="width:15%">Tags</th>
            </tr>
          </thead>
          <tbody id="rows"></tbody>
        </table>
      </div>

      <div class="footer">
        <div class="count" id="count">0 results</div>
        <div class="pager">
          <button class="btn" id="prev">Prev</button>
          <span class="muted" id="pageInfo"></span>
          <button class="btn" id="next">Next</button>
        </div>
      </div>
    </div>
    <p class="muted" style="margin-top:10px">Data loaded from <code>paul-stein-content1.csv</code>. Update that file to change records. Supports ~5–10k rows comfortably; 500 is trivial.</p>
  </div>

  <!-- Lightweight fuzzy search (Fuse.js) via CDN -->
  <script src="https://cdn.jsdelivr.net/npm/fuse.js@7.0.0"></script>
  <script>
  // ===== Configuration =====
  const DATA_URL = 'paul-stein-content1.csv'; // same-repo CSV file
  const KEYS = ['name','description','group','tags']; // fields to search
  const FACET_FIELD = 'group'; // quick-filter chips

  // ===== State =====
  let raw = [];
  let filtered = [];
  let fuse = null;
  let pageSize = 25;
  let page = 1;
  let activeChip = '';

  // ===== Elements =====
  const qEl = document.getElementById('q');
  const rowsEl = document.getElementById('rows');
  const countEl = document.getElementById('count');
  const prevEl = document.getElementById('prev');
  const nextEl = document.getElementById('next');
  const pageInfoEl = document.getElementById('pageInfo');
  const facetsEl = document.getElementById('facets');
  const groupEl = document.getElementById('group');
  const pageSizeEl = document.getElementById('pageSize');

  // ===== Helpers =====
  async function loadData(){
    const res = await fetch(DATA_URL, {cache:'no-store'});
    if(!res.ok) throw new Error('Failed to load data.csv');
    const csv = await res.csv();
    raw = csv.records || csv; // allow either [{...}] or {records:[...]}
    // normalize tags to array
    raw.forEach(r=>{ if(typeof r.tags==='string') r.tags = r.tags.split(',').map(s=>s.trim()).filter(Boolean); r.tags ||= []; });
    fuse = new Fuse(raw, {keys: KEYS, threshold: 0.35, ignoreLocation:true, minMatchCharLength:2});
    initFacets();
    apply();
  }

  function initFacets(){
    // build facet chips from unique groups
    const groups = [...new Set(raw.map(r=>r[FACET_FIELD]).filter(Boolean))].sort();
    facetsEl.innerHTML = '';
    groups.slice(0,12).forEach(g=>{
      const b = document.createElement('button');
      b.className = 'chip';
      b.textContent = g;
      b.onclick = ()=>{ activeChip = (activeChip===g?'':g); updateChips(); apply(); };
      facetsEl.appendChild(b);
    });
    // also populate select dropdown for group
    groupEl.innerHTML = '<option value="">All groups</option>' + groups.map(g=>`<option>${escapeHtml(g)}</option>`).join('');
  }

  function updateChips(){
    [...facetsEl.children].forEach(ch=>{
      ch.classList.toggle('active', ch.textContent===activeChip);
    });
    // sync dropdown when a chip toggles
    groupEl.value = activeChip || '';
  }

  function apply(){
    // text search
    const q = qEl.value.trim();
    let list = q ? fuse.search(q).map(r=>r.item) : raw.slice();

    // chip / dropdown filter
    const group = groupEl.value || activeChip || '';
    if(group) list = list.filter(r => (r[FACET_FIELD]||'') === group);

    filtered = list;
    page = 1; // reset page on new search/filter
    render();
  }

  function render(){
    const total = filtered.length;
    const pages = Math.max(1, Math.ceil(total / pageSize));
    if(page>pages) page = pages;
    const start = (page-1)*pageSize;
    const end = Math.min(start+pageSize, total);

    rowsEl.innerHTML = filtered.slice(start, end).map(rowToTr).join('');
    countEl.textContent = `${total.toLocaleString()} result${total===1?'':'s'}${total?` • showing ${start+1}-${end}`:''}`;
    pageInfoEl.textContent = `${page} / ${pages}`;
    prevEl.disabled = (page<=1);
    nextEl.disabled = (page>=pages);
  }

  function rowToTr(r){
    const tags = (r.tags||[]).map(t=>`<span class="tag">${escapeHtml(t)}</span>`).join('');
    const name = r.url ? `<a href="${encodeURI(r.url)}" target="_blank" rel="noopener">${escapeHtml(r.name||'Untitled')}</a>` : escapeHtml(r.name||'Untitled');
    return `<tr>
      <td>${name}<div class="muted">${escapeHtml(r.id||'')}</div></td>
      <td>${escapeHtml(r.description||'')}</td>
      <td>${escapeHtml(r.group||'')}</td>
      <td>${tags}</td>
    </tr>`;
  }

  function escapeHtml(s){ return String(s).replace(/[&<>"]+/g, m=>({"&":"&amp;","<":"&lt;",">":"&gt;","\"":"&quot;"}[m])); }

  // ===== Events =====
  qEl.addEventListener('input', ()=> apply());
  groupEl.addEventListener('change', ()=> { activeChip = groupEl.value || ''; updateChips(); apply(); });
  pageSizeEl.addEventListener('change', ()=> { pageSize = parseInt(pageSizeEl.value,10)||25; render(); });
  prevEl.addEventListener('click', ()=> { if(page>1){ page--; render(); }});
  nextEl.addEventListener('click', ()=> { page++; render(); });

  loadData().catch(err=>{
    rowsEl.innerHTML = `<tr><td colspan="4" class="muted">${escapeHtml(err.message)}</td></tr>`;
  });
  </script>
</body>
</html>
