<script>
  import ConfirmDialog from './ConfirmDialog.svelte'
  import { addTask, addTemplate, removeTemplate, updateTemplate, toggleFavorite, togglePin, recordTemplateUse, archiveTemplate, duplicateTemplate } from './taskStore.svelte.js'
  import { templates } from './taskStore.svelte.js'
  import { Plus, X, Copy, Search, ArrowUpDown, Star, Pin, PinOff, StarOff, Archive, ArchiveRestore, CopyPlus, Trash2, CheckSquare, Square, Download, Upload, Pencil, Check, FileText, ListChecks, ShoppingCart, Calendar, Dumbbell, BookOpen, Target, Lightbulb, Sun, Moon, Sparkles, Briefcase, Heart, Zap, Coffee, Clock, AlarmCheck, List, Notebook, Palette, Leaf, Flame, Trophy, Rocket, Compass, Camera, Headphones, Globe, Bell, MessageSquare, HelpCircle } from 'lucide-svelte'

  let search = $state('')
  let sort = $state('name')
  let filter = $state('all')
  let selectMode = $state(false)
  let selected = $state(new Set())
  let editingId = $state(null)
  let editTitle = $state('')

  let deleteId = $state(null)
  let deleteLabel = $state('')
  let archiveId = $state(null)

  let toasts = $state([])
  let toastId = $state(0)

  const ICON_NAMES = ['FileText','ListChecks','ShoppingCart','Calendar','Dumbbell','BookOpen','Target','Lightbulb','Sun','Moon','Sparkles','Briefcase','Heart','Zap','Coffee','Clock','AlarmCheck','List','Notebook','Palette','Leaf','Flame','Trophy','Rocket','Compass','Camera','Headphones','Globe','Bell','MessageSquare']
  const ICON_MAP = { FileText, ListChecks, ShoppingCart, Calendar, Dumbbell, BookOpen, Target, Lightbulb, Sun, Moon, Sparkles, Briefcase, Heart, Zap, Coffee, Clock, AlarmCheck, List, Notebook, Palette, Leaf, Flame, Trophy, Rocket, Compass, Camera, Headphones, Globe, Bell, MessageSquare }
  const COLORS = ['', '#ef4444','#f97316','#eab308','#22c55e','#14b8a6','#3b82f6','#8b5cf6','#ec4899','#64748b']

  let sortDir = $state(1)
  let showCreate = $state(false)
  let createName = $state('')
  let createIcon = $state('')
  let createColor = $state('')
  let showAllIcons = $state(false)
  let showHelp = $state(false)

  function openCreate() {
    createName = ''
    createIcon = ''
    createColor = ''
    showCreate = true
  }

  function confirmCreate() {
    if (!createName.trim()) return
    addTemplate({ title: createName.trim(), icon: createIcon, color: createColor, items: [] })
    showCreate = false
    toast('Template created', 'success')
  }

  function toast(msg, type = 'success') {
    const id = ++toastId
    toasts = [...toasts, { id, msg, type }]
    setTimeout(() => { toasts = toasts.filter(t => t.id !== id) }, 3000)
  }

  let filtered = $derived.by(() => {
    let list = templates.items.filter(t => !t.archived)
    if (filter === 'favorites') list = list.filter(t => t.favorite)
    if (filter === 'pinned') list = list.filter(t => t.pinned)
    if (search.trim()) {
      const q = search.toLowerCase()
      list = list.filter(t => t.title.toLowerCase().includes(q))
    }
    const dir = sortDir
    if (sort === 'name') list = [...list].sort((a, b) => dir * a.title.localeCompare(b.title))
    if (sort === 'uses') list = [...list].sort((a, b) => dir * ((b.useCount || 0) - (a.useCount || 0)))
    if (sort === 'recent') list = [...list].sort((a, b) => dir * ((b.lastUsed || 0) - (a.lastUsed || 0)))
    if (sort === 'newest') list = [...list].sort((a, b) => dir * ((b.createdAt || 0) - (a.createdAt || 0)))
    return list
  })

  let recentlyUsed = $derived(filtered.filter(t => t.lastUsed).sort((a, b) => (b.lastUsed || 0) - (a.lastUsed || 0)).slice(0, 5))
  let allList = $derived(filtered)

  function toggleSort(col) {
    if (sort === col) { sortDir *= -1 } else { sort = col; sortDir = 1 }
  }

  function toggleSelectAll() {
    if (selected.size === allList.length) { selected = new Set() } else { selected = new Set(allList.map(t => t.id)) }
  }

  function bulkDelete() {
    for (const id of selected) removeTemplate(id)
    toast(`Deleted ${selected.size} template${selected.size > 1 ? 's' : ''}`, 'success')
    selected = new Set()
    selectMode = false
  }

  function doDelete() {
    if (deleteLabel === 'archive') {
      archiveTemplate(deleteId)
      toast('Template archived', 'success')
    } else {
      removeTemplate(deleteId)
      toast('Template deleted', 'success')
    }
    deleteId = null
    deleteLabel = ''
  }

  function startEdit(id, title) {
    editingId = id; editTitle = title
  }

  function saveEdit(id) {
    if (editTitle.trim()) updateTemplate(id, { title: editTitle.trim() })
    editingId = null; editTitle = ''
  }

  function handleUse(t) {
    recordTemplateUse(t.id)
    addTask(t.title, '', '', null, null, null, null, [], null, t.items || [])
    toast('Task added to Today', 'success')
  }

  function handleDuplicate(t) {
    duplicateTemplate(t.id)
    toast('Template duplicated', 'success')
  }

  function handleToggleArchive(t) {
    if (t.archived) {
      archiveTemplate(t.id)
      toast('Template restored', 'success')
    } else {
      archiveId = t.id
      deleteLabel = 'archive'
      deleteId = t.id
    }
  }

  function exportTemplates() {
    const json = JSON.stringify(templates.items, null, 2)
    const blob = new Blob([json], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url; a.download = `templates-backup.json`; a.click()
    URL.revokeObjectURL(url)
    toast('Templates exported', 'success')
  }

  function importTemplates() {
    const input = document.createElement('input')
    input.type = 'file'; input.accept = '.json'
    input.onchange = async () => {
      try {
        const text = await input.files[0].text()
        const data = JSON.parse(text)
        if (!Array.isArray(data)) throw new Error('Invalid')
        for (const t of data) addTemplate({ title: t.title, items: t.items || [], icon: t.icon || '', color: t.color || '' })
        toast(`Imported ${data.length} templates`, 'success')
      } catch { toast('Invalid file', 'error') }
    }
    input.click()
  }

  function cycleIcon(id, current) {
    const idx = ICON_NAMES.indexOf(current)
    const next = ICON_NAMES[(idx + 1) % ICON_NAMES.length]
    updateTemplate(id, { icon: next })
  }

  function cycleColor(id, current) {
    const idx = COLORS.indexOf(current)
    const next = COLORS[(idx + 1) % COLORS.length]
    updateTemplate(id, { color: next })
  }
</script>

<div class="view-content">
  <div class="header-row">
    <h2 class="view-title">Templates <span class="count">{templates.items.filter(t => !t.archived).length}</span></h2>
    <div class="header-actions-tpl">
      <button class="tpl-btn" onclick={() => selectMode = !selectMode} title="Select multiple" class:active={selectMode}>
        <CheckSquare size={14} strokeWidth={1.5} />
      </button>
      <button class="tpl-btn" onclick={importTemplates} title="Import templates">
        <Upload size={14} strokeWidth={1.5} />
      </button>
      <button class="tpl-btn" onclick={exportTemplates} title="Export templates">
        <Download size={14} strokeWidth={1.5} />
      </button>
      <button class="tpl-btn" class:active={showHelp} onclick={() => showHelp = !showHelp} title="How this page works">
        <HelpCircle size={14} strokeWidth={1.5} />
      </button>
    </div>
  </div>

  {#if showHelp}
    <div class="help-panel">
      <div class="help-item"><CheckSquare size={13} strokeWidth={1.5} /> Select: checkbox icon to bulk select templates</div>
      <div class="help-item"><Star size={13} strokeWidth={1.5} /> Favorite: star a template for quick access</div>
      <div class="help-item"><Pin size={13} strokeWidth={1.5} /> Pin: keep a template at the top of the list</div>
      <div class="help-item"><Pencil size={13} strokeWidth={1.5} /> Rename: click a name to edit it inline</div>
      <div class="help-item"><Copy size={13} strokeWidth={1.5} /> Duplicate: clone a template with one click</div>
      <div class="help-item"><Archive size={13} strokeWidth={1.5} /> Archive: hide without deleting (restore later)</div>
      <div class="help-item"><Search size={13} strokeWidth={1.5} /> Search: filter by name or keyword</div>
      <div class="help-item"><ArrowUpDown size={13} strokeWidth={1.5} /> Sort: by name, use count, recency, or date</div>
    </div>
  {/if}

  <div class="search-bar">
    <Search size={14} strokeWidth={1.5} class="search-icon" />
    <input type="text" class="search-input" placeholder="Search templates..." bind:value={search} />
    {#if search}
      <button class="search-clear" onclick={() => search = ''}>
        <X size={14} strokeWidth={1.5} />
      </button>
    {/if}
  </div>

  <div class="toolbar">
    <div class="filter-tabs">
      {#each ['all','favorites','pinned'] as f}
        <button class="filter-tab" class:active={filter === f} onclick={() => { filter = f; selected = new Set() }}>
          {f === 'all' ? 'All' : f === 'favorites' ? 'Favorites' : 'Pinned'}
        </button>
      {/each}
    </div>
    <div class="sort-group">
      <span class="sort-label">Sort:</span>
      <select class="sort-select" bind:value={sort} onchange={() => sortDir = 1}>
        <option value="name">Name</option>
        <option value="uses">Use count</option>
        <option value="recent">Recently used</option>
        <option value="newest">Newest</option>
      </select>
      <button class="sort-dir-btn" onclick={() => sortDir *= -1} title="Toggle order">
        <ArrowUpDown size={14} strokeWidth={1.5} />
      </button>
    </div>
  </div>

  {#if templates.items.filter(t => !t.archived).length === 0}
    <div class="empty">
      <div class="empty-icon"><FileText size={36} strokeWidth={1.5} /></div>
      <p class="empty-title">No templates yet</p>
      <p class="empty-sub">Create a template for any routine or recurring task. Use them to add pre-built task lists to your day instantly.</p>
      <button class="empty-cta" onclick={openCreate}>
        <Plus size={16} strokeWidth={1.5} /> Create your first template
      </button>
    </div>
  {:else}
    {#if selectMode}
      <div class="select-bar">
          <button class="select-all-btn" onclick={toggleSelectAll}>
            {#if selected.size === allList.length}
              <Square size={14} strokeWidth={1.5} />
            {:else}
              <CheckSquare size={14} strokeWidth={1.5} />
            {/if}
            {selected.size === allList.length ? 'Deselect all' : 'Select all'}
          </button>
        <span class="select-count">{selected.size} selected</span>
        {#if selected.size > 0}
          <button class="select-delete-btn" onclick={bulkDelete}>
            <Trash2 size={14} strokeWidth={1.5} /> Delete selected
          </button>
        {/if}
      </div>
    {/if}

    {#if recentlyUsed.length > 0 && !search && filter === 'all'}
      <div class="section">
        <div class="section-header">
          <span class="section-title">Recently used</span>
        </div>
        <div class="template-list">
          {#each recentlyUsed as t (t.id)}
            <div class="template-row" class:selected={selected.has(t.id)}>
              {#if selectMode}
                <button class="tpl-check" onclick={() => { const s = new Set(selected); if (s.has(t.id)) s.delete(t.id); else s.add(t.id); selected = s }}>
                  {#if selected.has(t.id)}<CheckSquare size={16} strokeWidth={1.5} />{:else}<Square size={16} strokeWidth={1.5} />{/if}
                </button>
              {/if}
              <button class="tpl-icon-btn" onclick={() => cycleIcon(t.id, t.icon || '')} title="Change icon">
                <span class="tpl-icon">
                  {#if t.icon && ICON_MAP[t.icon]}
                    <svelte:component this={ICON_MAP[t.icon]} size={16} strokeWidth={1.5} />
                  {:else}
                    <FileText size={16} strokeWidth={1.5} />
                  {/if}
                </span>
              </button>
              <button class="tpl-color-dot" style="background: {t.color || 'var(--border)'}" onclick={() => cycleColor(t.id, t.color || '')} title="Change color">
              </button>
              <div class="tpl-info">
                {#if editingId === t.id}
                  <input type="text" class="tpl-edit-input" bind:value={editTitle} onkeydown={(e) => { if (e.key === 'Enter') saveEdit(t.id); if (e.key === 'Escape') editingId = null }} autofocus />
                  <button class="tpl-save-btn" onclick={() => saveEdit(t.id)}>
                    <Check size={12} strokeWidth={2} />
                  </button>
                {:else}
                  <span class="tpl-name" ondblclick={() => startEdit(t.id, t.title)}>{t.title}</span>
                  {#if t.useCount > 0}
                    <span class="tpl-uses">{t.useCount}×</span>
                  {/if}
                {/if}
              </div>
              <div class="tpl-actions">
                <button class="tpl-act" onclick={() => handleUse(t)} title="Create task">
                  <Copy size={13} strokeWidth={1.5} />
                </button>
                <button class="tpl-act" onclick={() => handleDuplicate(t)} title="Duplicate">
                  <CopyPlus size={13} strokeWidth={1.5} />
                </button>
                <button class="tpl-act" class:active={t.favorite} onclick={() => toggleFavorite(t.id)} title={t.favorite ? 'Unfavorite' : 'Favorite'}>
                  {#if t.favorite}<Star size={13} strokeWidth={1.5} />{:else}<StarOff size={13} strokeWidth={1.5} />{/if}
                </button>
                <button class="tpl-act" class:active={t.pinned} onclick={() => togglePin(t.id)} title={t.pinned ? 'Unpin' : 'Pin'}>
                  {#if t.pinned}<Pin size={13} strokeWidth={1.5} />{:else}<PinOff size={13} strokeWidth={1.5} />{/if}
                </button>
                <button class="tpl-act" onclick={() => handleToggleArchive(t)} title="Archive">
                  <Archive size={13} strokeWidth={1.5} />
                </button>
                <button class="tpl-act del" onclick={() => { deleteId = t.id; deleteLabel = 'delete' }} title="Delete">
                  <Trash2 size={13} strokeWidth={1.5} />
                </button>
              </div>
            </div>
          {/each}
        </div>
      </div>
    {/if}

    <div class="section">
      <div class="section-header">
        <span class="section-title">All templates</span>
        <span class="section-count">{allList.length}</span>
      </div>
      <div class="template-list">
        {#each allList as t (t.id)}
          <div class="template-row" class:selected={selected.has(t.id)}>
            {#if selectMode}
              <button class="tpl-check" onclick={() => { const s = new Set(selected); if (s.has(t.id)) s.delete(t.id); else s.add(t.id); selected = s }}>
                {#if selected.has(t.id)}<CheckSquare size={16} strokeWidth={1.5} />{:else}<Square size={16} strokeWidth={1.5} />{/if}
              </button>
            {/if}
            <button class="tpl-icon-btn" onclick={() => cycleIcon(t.id, t.icon || '')} title="Change icon">
                <span class="tpl-icon">
                  {#if t.icon && ICON_MAP[t.icon]}
                    <svelte:component this={ICON_MAP[t.icon]} size={16} strokeWidth={1.5} />
                  {:else}
                    <FileText size={16} strokeWidth={1.5} />
                  {/if}
                </span>
              </button>
              <button class="tpl-color-dot" style="background: {t.color || 'var(--border)'}" onclick={() => cycleColor(t.id, t.color || '')} title="Change color">
              </button>
              <div class="tpl-info">
              {#if editingId === t.id}
                <input type="text" class="tpl-edit-input" bind:value={editTitle} onkeydown={(e) => { if (e.key === 'Enter') saveEdit(t.id); if (e.key === 'Escape') editingId = null }} autofocus />
                <button class="tpl-save-btn" onclick={() => saveEdit(t.id)}>
                  <Check size={12} strokeWidth={2} />
                </button>
              {:else}
                <span class="tpl-name" ondblclick={() => startEdit(t.id, t.title)}>{t.title}</span>
                {#if t.pinned}<span class="tpl-badge pinned-badge">P</span>{/if}
                {#if t.favorite}<span class="tpl-badge fav-badge">★</span>{/if}
                {#if t.useCount > 0}
                  <span class="tpl-uses">{t.useCount}×</span>
                {/if}
              {/if}
            </div>
            <div class="tpl-actions">
              <button class="tpl-act" onclick={() => handleUse(t)} title="Create task">
                <Copy size={13} strokeWidth={1.5} />
              </button>
              <button class="tpl-act" onclick={() => handleDuplicate(t)} title="Duplicate">
                <CopyPlus size={13} strokeWidth={1.5} />
              </button>
              <button class="tpl-act" class:active={t.favorite} onclick={() => toggleFavorite(t.id)} title={t.favorite ? 'Unfavorite' : 'Favorite'}>
                {#if t.favorite}<Star size={13} strokeWidth={1.5} />{:else}<StarOff size={13} strokeWidth={1.5} />{/if}
              </button>
              <button class="tpl-act" class:active={t.pinned} onclick={() => togglePin(t.id)} title={t.pinned ? 'Unpin' : 'Pin'}>
                {#if t.pinned}<Pin size={13} strokeWidth={1.5} />{:else}<PinOff size={13} strokeWidth={1.5} />{/if}
              </button>
              <button class="tpl-act" onclick={() => handleToggleArchive(t)} title="Archive">
                <Archive size={13} strokeWidth={1.5} />
              </button>
              <button class="tpl-act del" onclick={() => { deleteId = t.id; deleteLabel = 'delete' }} title="Delete">
                <Trash2 size={13} strokeWidth={1.5} />
              </button>
            </div>
          </div>
        {/each}
      </div>
    </div>

    <div class="add-row">
      <button class="add-btn" onclick={openCreate}>
        <Plus size={16} strokeWidth={1.5} /> New template
      </button>
    </div>
  {/if}
</div>

<div class="toast-container">
  {#each toasts as t (t.id)}
    <div class="toast" class:toast-error={t.type === 'error'}>
      <span>{t.msg}</span>
    </div>
  {/each}
</div>

{#if showCreate}
  <div class="create-overlay" role="dialog" onclick={() => showCreate = false} onkeydown={(e) => { if (e.key === 'Escape') showCreate = false }} tabindex="0">
    <div class="create-dialog" onclick={(e) => e.stopPropagation()} role="document">
      <div class="create-dialog-header">
        <span class="create-dialog-title">New template</span>
        <button class="create-close-btn" onclick={() => showCreate = false}>
          <X size={16} strokeWidth={1.5} />
        </button>
      </div>
      <div class="create-dialog-body">
        <div class="create-field">
          <label class="create-label">Name</label>
          <input type="text" class="create-input" placeholder="Template name" bind:value={createName} onkeydown={(e) => { if (e.key === 'Enter') confirmCreate() }} autofocus />
        </div>
        <div class="create-field">
          <label class="create-label">Icon</label>
          <div class="icon-grid">
            <button class="icon-opt" class:selected={!createIcon} onclick={() => createIcon = ''} title="No icon" style={!createIcon && createColor ? `background:${createColor}22; border-color:${createColor}66` : ''}>
              <FileText size={16} strokeWidth={1.5} />
            </button>
            {#each ICON_NAMES.slice(0, 7) as name}
              <button class="icon-opt" class:selected={createIcon === name} onclick={() => createIcon = name} title={name} style={createIcon === name && createColor ? `background:${createColor}22; border-color:${createColor}66` : ''}>
                <svelte:component this={ICON_MAP[name]} size={16} strokeWidth={1.5} />
              </button>
            {/each}
            {#if showAllIcons}
              {#each ICON_NAMES.slice(7) as name}
                <button class="icon-opt" class:selected={createIcon === name} onclick={() => createIcon = name} title={name} style={createIcon === name && createColor ? `background:${createColor}22; border-color:${createColor}66` : ''}>
                  <svelte:component this={ICON_MAP[name]} size={16} strokeWidth={1.5} />
                </button>
              {/each}
              <button class="icon-opt icon-less" onclick={() => showAllIcons = false} title="Show less">
                <span style="font-size:16px;font-weight:600;line-height:1">−</span>
              </button>
            {:else}
              <button class="icon-opt icon-more" onclick={() => showAllIcons = true} title="Show all icons">
                <span style="font-size:16px;font-weight:600;line-height:1">+</span>
              </button>
            {/if}
          </div>
        </div>
        <div class="create-field">
          <label class="create-label">Color</label>
          <div class="color-row">
            <button class="color-opt" class:selected={!createColor} onclick={() => createColor = ''} title="No color" style="background: var(--border)">
              <X size={10} strokeWidth={2} />
            </button>
            {#each COLORS as c}
              {#if c}
                <button class="color-opt" class:selected={createColor === c} onclick={() => createColor = c} title={c} style="background: {c}"></button>
              {/if}
            {/each}
          </div>
        </div>
      </div>
      <div class="create-dialog-footer">
        <button class="dlg-btn-secondary" onclick={() => showCreate = false}>Cancel</button>
        <button class="dlg-btn-primary" onclick={confirmCreate} disabled={!createName.trim()}>Create</button>
      </div>
    </div>
  </div>
{/if}

<ConfirmDialog
  open={deleteId !== null}
  title={deleteLabel === 'archive' ? 'Archive template' : 'Delete template'}
  message={deleteLabel === 'archive' ? 'Archive this template? You can restore it later.' : 'Delete this template permanently? This cannot be undone.'}
  confirmLabel={deleteLabel === 'archive' ? 'Archive' : 'Delete'}
  onConfirm={doDelete}
  onCancel={() => { deleteId = null; deleteLabel = ''; archiveId = null }}
/>

<style>
  .view-content { padding: 24px 28px; flex: 1; overflow-y: auto; display: flex; flex-direction: column; gap: 0; }

  .header-row { display: flex; align-items: center; justify-content: space-between; margin-bottom: 16px; }
  .view-title { font-size: 22px; font-weight: 700; color: var(--text); margin: 0; display: flex; align-items: center; gap: 8px; }
  .count { font-size: 12px; font-weight: 600; color: var(--text-muted); background: var(--surface); padding: 2px 10px; border-radius: 20px; border: 1px solid var(--border); }
  .header-actions-tpl { display: flex; gap: 4px; }
  .tpl-btn { width: 32px; height: 32px; border-radius: 8px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--text-muted); background: transparent; border: none; padding: 0; transition: all 0.15s var(--ease); }
  .tpl-btn:hover { color: var(--text); background: var(--surface-hover); }
  .tpl-btn.active { color: var(--accent); background: var(--accent-subtle); }

  .search-bar { position: relative; margin-bottom: 12px; }
  .search-icon { position: absolute; left: 12px; top: 50%; transform: translateY(-50%); color: var(--text-muted); pointer-events: none; }
  .search-input { width: 100%; padding: 10px 36px 10px 36px; font-size: 13px; border-radius: var(--radius-md); border: 1px solid var(--border); background: var(--surface); color: var(--text); box-sizing: border-box; outline: none; transition: border-color 0.2s var(--ease); }
  .search-input:focus { border-color: var(--accent); box-shadow: var(--accent-ring); }
  .search-input::placeholder { color: var(--text-muted); }
  .search-clear { position: absolute; right: 8px; top: 50%; transform: translateY(-50%); width: 24px; height: 24px; border-radius: 6px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--text-muted); background: transparent; border: none; padding: 0; }
  .search-clear:hover { color: var(--text); background: var(--surface-hover); }

  .help-panel { display: flex; flex-wrap: wrap; gap: 6px 18px; padding: 12px 16px; background: var(--surface); border: 1px solid var(--border); border-radius: var(--radius-md); margin-bottom: 12px; font-size: 12px; color: var(--text-secondary); animation: fadeIn 0.2s var(--ease-out); }
  .help-item { display: flex; align-items: center; gap: 6px; white-space: nowrap; }
  @keyframes fadeIn { from { opacity: 0; transform: translateY(-6px); } to { opacity: 1; transform: translateY(0); } }

  .toolbar { display: flex; align-items: center; justify-content: space-between; margin-bottom: 16px; gap: 8px; flex-wrap: wrap; }
  .filter-tabs { display: flex; gap: 2px; background: var(--surface); border-radius: var(--radius-md); padding: 2px; border: 1px solid var(--border); }
  .filter-tab { padding: 5px 14px; font-size: 12px; font-weight: 500; color: var(--text-muted); background: transparent; border: none; border-radius: 6px; cursor: pointer; transition: all 0.15s var(--ease); white-space: nowrap; }
  .filter-tab:hover { color: var(--text); }
  .filter-tab.active { color: var(--text); background: var(--bg); box-shadow: 0 1px 3px rgba(0,0,0,0.06); }
  .sort-group { display: flex; align-items: center; gap: 6px; }
  .sort-label { font-size: 11px; color: var(--text-muted); font-weight: 500; }
  .sort-select { padding: 5px 8px; font-size: 12px; border-radius: 6px; border: 1px solid var(--border); background: var(--surface); color: var(--text); outline: none; cursor: pointer; }
  .sort-select:focus { border-color: var(--accent); }
  .sort-dir-btn { width: 28px; height: 28px; border-radius: 6px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--text-muted); background: transparent; border: none; padding: 0; transition: all 0.15s var(--ease); }
  .sort-dir-btn:hover { color: var(--text); background: var(--surface-hover); }

  .select-bar { display: flex; align-items: center; gap: 12px; padding: 8px 12px; background: var(--surface); border-radius: var(--radius-md); border: 1px solid var(--border); margin-bottom: 12px; }
  .select-all-btn { display: flex; align-items: center; gap: 6px; padding: 4px 10px; font-size: 12px; font-weight: 500; color: var(--text-secondary); background: transparent; border: none; border-radius: 6px; cursor: pointer; }
  .select-all-btn:hover { background: var(--surface-hover); }
  .select-count { font-size: 12px; color: var(--text-muted); flex: 1; }
  .select-delete-btn { display: flex; align-items: center; gap: 6px; padding: 5px 12px; font-size: 12px; font-weight: 600; color: var(--danger); background: rgba(192,112,96,0.1); border: 1px solid rgba(192,112,96,0.2); border-radius: 6px; cursor: pointer; transition: all 0.15s var(--ease); }
  .select-delete-btn:hover { background: rgba(192,112,96,0.2); }

  .section { margin-bottom: 4px; }
  .section-header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 6px; }
  .section-title { font-size: 11px; font-weight: 600; color: var(--text-muted); text-transform: uppercase; letter-spacing: 0.5px; }
  .section-count { font-size: 11px; color: var(--text-muted); }

  .template-list { display: flex; flex-direction: column; gap: 2px; }
  .template-row { display: flex; align-items: center; gap: 8px; padding: 8px 10px; border-radius: var(--radius-md); transition: all 0.15s var(--ease); cursor: default; }
  .template-row:hover { background: var(--surface-hover); }
  .template-row.selected { background: var(--accent-subtle); }

  .tpl-check { width: 28px; height: 28px; border-radius: 6px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--text-muted); background: transparent; border: none; padding: 0; flex-shrink: 0; }
  .tpl-check:hover { color: var(--accent); }

  .tpl-icon-btn { width: 32px; height: 32px; border-radius: 8px; display: flex; align-items: center; justify-content: center; cursor: pointer; background: var(--surface); border: 1px solid var(--border); padding: 0; flex-shrink: 0; transition: all 0.15s var(--ease); }
  .tpl-icon-btn:hover { border-color: var(--accent-subtle); transform: scale(1.05); }
  .tpl-icon { display: flex; align-items: center; justify-content: center; color: var(--text-secondary); }

  .tpl-color-dot { width: 10px; height: 10px; border-radius: 50%; border: 1.5px solid var(--border); flex-shrink: 0; cursor: pointer; transition: all 0.15s var(--ease); }
  .tpl-color-dot:hover { transform: scale(1.3); }

  .tpl-info { flex: 1; min-width: 0; display: flex; align-items: center; gap: 6px; }
  .tpl-name { font-size: 13px; font-weight: 500; color: var(--text); overflow: hidden; text-overflow: ellipsis; white-space: nowrap; cursor: text; }
  .tpl-edit-input { flex: 1; padding: 4px 8px; font-size: 13px; border-radius: 6px; border: 1px solid var(--accent); background: var(--bg); color: var(--text); outline: none; min-width: 60px; }
  .tpl-save-btn { width: 24px; height: 24px; border-radius: 6px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--complete); background: var(--complete-bg); border: none; padding: 0; flex-shrink: 0; }
  .tpl-save-btn:hover { filter: brightness(1.1); }
  .tpl-badge { font-size: 9px; font-weight: 700; padding: 1px 5px; border-radius: 4px; }
  .pinned-badge { color: var(--accent); background: var(--accent-subtle); }
  .fav-badge { color: #eab308; background: rgba(234,179,8,0.15); }
  .tpl-uses { font-size: 10px; font-weight: 600; color: var(--text-muted); background: var(--surface); padding: 1px 6px; border-radius: 10px; }

  .tpl-actions { display: flex; gap: 1px; flex-shrink: 0; }
  .tpl-act { width: 28px; height: 28px; border-radius: 6px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--text-muted); background: transparent; border: none; padding: 0; transition: all 0.15s var(--ease); }
  .tpl-act:hover { color: var(--text); background: var(--surface); }
  .tpl-act.active { color: #eab308; }
  .tpl-act.active:hover { background: rgba(234,179,8,0.1); }
  .tpl-act.del:hover { color: var(--danger); background: rgba(192,112,96,0.1); }

  .add-row { margin-top: 12px; }
  .add-btn { display: flex; align-items: center; gap: 6px; padding: 8px 16px; font-size: 13px; font-weight: 600; color: var(--accent); background: var(--accent-subtle); border: 1px dashed var(--accent); border-radius: var(--radius-md); cursor: pointer; transition: all 0.15s var(--ease); width: 100%; justify-content: center; }
  .add-btn:hover { background: rgba(var(--accent-rgb,99,102,241),0.15); }

  .empty { display: flex; flex-direction: column; align-items: center; justify-content: center; padding: 48px 24px; text-align: center; flex: 1; }
  .empty-icon { margin-bottom: 12px; opacity: 0.4; color: var(--text-muted); }
  .empty-title { font-size: 16px; font-weight: 600; color: var(--text); margin: 0 0 6px; }
  .empty-sub { font-size: 13px; color: var(--text-muted); max-width: 340px; line-height: 1.6; margin: 0 0 20px; }
  .empty-cta { display: inline-flex; align-items: center; gap: 6px; padding: 10px 20px; font-size: 13px; font-weight: 600; color: #fff; background: var(--accent); border: none; border-radius: var(--radius-md); cursor: pointer; transition: all 0.15s var(--ease); }
  .empty-cta:hover { filter: brightness(1.1); transform: translateY(-1px); }

  .toast-container { position: fixed; bottom: 20px; left: 50%; transform: translateX(-50%); display: flex; flex-direction: column; gap: 6px; z-index: 999; pointer-events: none; }
  .toast { padding: 10px 20px; border-radius: var(--radius-md); background: var(--surface-raised); border: 1px solid var(--border); box-shadow: var(--shadow-lg); font-size: 13px; font-weight: 500; color: var(--text); animation: toastIn 0.25s var(--ease-out); pointer-events: auto; }
  .toast-error { color: var(--danger); border-color: rgba(192,112,96,0.3); }
  @keyframes toastIn { from { opacity: 0; transform: translateY(12px); } to { opacity: 1; transform: translateY(0); } }

  .create-overlay { position: fixed; inset: 0; background: rgba(0,0,0,0.4); display: flex; align-items: center; justify-content: center; padding: 24px; z-index: 210; backdrop-filter: blur(16px); -webkit-backdrop-filter: blur(16px); }
  .create-dialog { background: var(--glass-bg); backdrop-filter: blur(var(--glass-blur)); border: 1px solid var(--glass-border); border-radius: var(--radius-xl); width: 100%; max-width: 440px; box-shadow: var(--shadow-xl); overflow: hidden; animation: scaleIn 0.2s var(--ease-out); }
  .create-dialog-header { display: flex; align-items: center; gap: 12px; padding: 16px 20px; border-bottom: 1px solid var(--glass-border); }
  .create-dialog-title { font-size: 15px; font-weight: 600; color: var(--text); flex: 1; }
  .create-close-btn { width: 32px; height: 32px; border-radius: var(--radius-sm); display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--text-muted); background: transparent; border: none; transition: all 0.2s var(--ease); flex-shrink: 0; }
  .create-close-btn:hover { background: var(--surface-hover); color: var(--text); transform: scale(1.05); }
  .create-dialog-body { padding: 20px; display: flex; flex-direction: column; gap: 16px; }
  .create-field { display: flex; flex-direction: column; gap: 6px; }
  .create-label { font-size: 12px; font-weight: 600; color: var(--text-secondary); text-transform: uppercase; letter-spacing: 0.4px; }
  .create-input { width: 100%; padding: 10px 14px; font-size: 13px; border-radius: var(--radius-md); border: 1px solid var(--border); background: var(--surface); color: var(--text); outline: none; transition: border-color 0.2s var(--ease); box-sizing: border-box; }
  .create-input:focus { border-color: var(--accent); box-shadow: var(--accent-ring); }
  .create-input::placeholder { color: var(--text-muted); }
  .icon-grid { display: flex; flex-wrap: wrap; gap: 4px; padding: 2px 0; }
  .icon-opt { width: 34px; height: 34px; border-radius: 8px; display: flex; align-items: center; justify-content: center; cursor: pointer; color: var(--text-secondary); background: var(--surface); border: 1px solid var(--border); padding: 0; transition: background 0.25s var(--ease), border-color 0.25s var(--ease), color 0.25s var(--ease); }
  .icon-opt:hover { border-color: var(--accent-subtle); color: var(--text); }
  .icon-opt.selected { border-color: var(--accent); color: var(--accent); background: var(--accent-subtle); }
  .icon-more { border: 1px dashed var(--accent); color: var(--accent); background: var(--accent-subtle); font-size: 16px; font-weight: 600; }
  .icon-more:hover { background: rgba(var(--accent-rgb,99,102,241),0.18); }
  .icon-less { border: 1px dashed var(--border); color: var(--text-muted); background: var(--surface); font-size: 16px; font-weight: 600; }
  .icon-less:hover { border-color: var(--text-muted); color: var(--text); }
  .color-row { display: flex; flex-wrap: wrap; gap: 6px; }
  .color-opt { width: 28px; height: 28px; border-radius: 50%; border: 2px solid var(--border); cursor: pointer; transition: all 0.1s var(--ease); display: flex; align-items: center; justify-content: center; padding: 0; }
  .color-opt:hover { transform: scale(1.15); }
  .color-opt.selected { border-color: var(--accent); box-shadow: 0 0 0 2px var(--bg), 0 0 0 4px var(--accent); }
  .create-dialog-footer { display: flex; gap: 10px; justify-content: flex-end; padding: 14px 20px; border-top: 1px solid var(--glass-border); }
  .dlg-btn-secondary { padding: 9px 20px; border-radius: var(--radius-md); font-size: 13px; font-weight: 500; cursor: pointer; background: var(--surface-raised); color: var(--text-secondary); border: 1px solid var(--glass-border); transition: all 0.2s var(--ease); }
  .dlg-btn-secondary:hover { background: var(--surface-hover); color: var(--text); }
  .dlg-btn-primary { padding: 9px 20px; border-radius: var(--radius-md); font-size: 13px; font-weight: 500; cursor: pointer; background: var(--accent); color: #fff; border: none; transition: all 0.2s var(--ease); }
  .dlg-btn-primary:hover { filter: brightness(1.1); }
  .dlg-btn-primary:disabled { opacity: 0.3; cursor: default; filter: none; }
</style>
