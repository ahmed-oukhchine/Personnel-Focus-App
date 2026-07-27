<script>
  import { X, Check, Sun, Moon, ChevronUp, ChevronDown, Flame, Clock, Copy, ChevronRight, Plus } from 'lucide-svelte'
  import { routines, addRoutine, removeRoutine, addRoutineItem, toggleRoutineItem, removeRoutineItem, moveRoutine, moveRoutineItem, duplicateRoutine, setRoutineItemTime, toggleRoutineCollapse, getRoutineStreak, getRoutineSummary, getRoutineTotalTime } from './taskStore.svelte.js'

  let title = $state(''), type = $state('morning'), itemInput = $state({}), timeEdit = $state({}), timeInput = $state({})
  let showCreate = $state(false), createName = $state(''), createType = $state('morning'), createTime = $state('')

  function resetRoutines() {
    const today = new Date().toISOString().split('T')[0]
    const lastReset = localStorage.getItem('focus-routine-reset')
    if (lastReset === today) return
    for (const r of routines.items) {
      for (const item of r.items) {
        if (item.completed) item.completed = false
      }
    }
    localStorage.setItem('focus-routine-reset', today)
  }

  resetRoutines()

  let summary = $derived(getRoutineSummary())

  function openCreate(t) {
    createType = t
    createName = ''
    createTime = ''
    showCreate = true
    setTimeout(() => document.querySelector('.dlg-name-input')?.focus(), 50)
  }

  function confirmCreate() {
    if (!createName.trim()) return
    addRoutine(createName.trim(), createType, createTime ? parseInt(createTime) : null)
    showCreate = false
  }

  function handleItemKey(e, rid) {
    if (e.key !== 'Enter') return
    const val = itemInput[rid]
    if (!val || !val.trim()) return
    addRoutineItem(rid, val.trim())
    itemInput = { ...itemInput, [rid]: '' }
  }

  function startTimeEdit(routineId, itemId, current) {
    timeEdit = { ...timeEdit, [`${routineId}-${itemId}`]: true }
    timeInput = { ...timeInput, [`${routineId}-${itemId}`]: current || '' }
    setTimeout(() => {
      const el = document.querySelector(`.te-input-${CSS.escape(routineId)}-${CSS.escape(itemId)}`)
      el?.focus()
      el?.select()
    }, 50)
  }

  function saveTime(routineId, itemId) {
    const val = timeInput[`${routineId}-${itemId}`]
    setRoutineItemTime(routineId, itemId, val || null)
    timeEdit = { ...timeEdit, [`${routineId}-${itemId}`]: false }
  }

  function handleTimeKey(e, routineId, itemId) {
    if (e.key === 'Enter') saveTime(routineId, itemId)
    if (e.key === 'Escape') timeEdit = { ...timeEdit, [`${routineId}-${itemId}`]: false }
  }

  function formatTime(min) {
    if (!min) return ''
    if (min < 60) return `${min}m`
    const h = Math.floor(min / 60)
    const m = min % 60
    return m ? `${h}h ${m}m` : `${h}h`
  }

  function visibleItems(r) {
    if (!r.collapseCompleted || !r.items.some(i => i.completed)) return r.items
    return r.items.filter(i => !i.completed)
  }

  function getPct(r) {
    if (!r.items.length) return 0
    return Math.round(r.items.filter(i => i.completed).length / r.items.length * 100)
  }
</script>

<div class="view-content">
  <h2 class="view-title">Routines</h2>
  <p class="view-sub">Morning and evening checklists</p>

  {#if routines.items.length > 0}
    <div class="summary-bar">
      <div class="summary-item"><Sun size={12} strokeWidth={1.5} /> Morning <span class="summary-count">{summary.morning.done}/{summary.morning.total}</span></div>
      <div class="summary-item"><Moon size={12} strokeWidth={1.5} /> Evening <span class="summary-count">{summary.evening.done}/{summary.evening.total}</span></div>
    </div>
  {/if}

  {#each ['morning', 'evening'] as routineType}
    {@const filtered = routines.items.filter(r => r.type === routineType)}
    <h3 class="section-heading">
      {#if routineType === 'morning'}<Sun size={14} strokeWidth={1.5} />{:else}<Moon size={14} strokeWidth={1.5} />{/if}
      {routineType === 'morning' ? 'Morning' : 'Evening'}
    </h3>
    {#if filtered.length === 0}
      <div class="section-empty">
        <div class="section-empty-icon">{#if routineType === 'morning'}<Sun size={22} strokeWidth={1.5} />{:else}<Moon size={22} strokeWidth={1.5} />{/if}</div>
        <p class="section-empty-text">No {routineType === 'morning' ? 'morning' : 'evening'} routines yet</p>
        <button class="section-empty-btn" onclick={() => openCreate(routineType)}>
          <Plus size={12} strokeWidth={1.5} /> Create {routineType === 'morning' ? 'morning' : 'evening'} routine
        </button>
      </div>
    {:else}
      {#each filtered as routine (routine.id)}
        {@const streak = getRoutineStreak(routine)}
        {@const totalTime = getRoutineTotalTime(routine)}
        {@const displayTime = routine.estimatedTime || totalTime}
        {@const doneCount = routine.items.filter(i => i.completed).length}
        {@const pct = getPct(routine)}
        {@const visItems = visibleItems(routine)}
        {@const hasCompleted = routine.items.some(i => i.completed)}
        <div class="card">
          <div class="card-header">
            <div class="card-header-left">
              <button class="card-move" onclick={() => moveRoutine(routine.id, 'up')} aria-label="Move up" tabindex="-1"><ChevronUp size={9} strokeWidth={1.5} /></button>
              <button class="card-move" onclick={() => moveRoutine(routine.id, 'down')} aria-label="Move down" tabindex="-1"><ChevronDown size={9} strokeWidth={1.5} /></button>
              <span class="card-title">{routine.title}</span>
              {#if streak > 0}
                <span class="badge badge-streak" aria-label="{streak} day streak"><Flame size={9} strokeWidth={1.5} /> {streak}d</span>
              {/if}
              {#if displayTime > 0}
                <span class="badge badge-time" aria-label="{formatTime(displayTime)} total"><Clock size={9} strokeWidth={1.5} /> {formatTime(displayTime)}</span>
              {/if}
            </div>
            <div class="card-header-right">
              <span class="card-progress" aria-label="{doneCount} of {routine.items.length} done">{doneCount}/{routine.items.length}</span>
              <button class="card-act" onclick={() => duplicateRoutine(routine.id)} aria-label="Copy Routine" title="Copy Routine"><Copy size={11} strokeWidth={1.5} /></button>
              <button class="card-act card-act-del" onclick={() => removeRoutine(routine.id)} aria-label="Delete Routine" title="Delete Routine">
                <X size={11} strokeWidth={1.5} />
              </button>
            </div>
          </div>
          {#if routine.items.length > 0}
            <div class="progress-track"><div class="progress-fill" style="width:{pct}%"></div></div>
          {/if}
          <div class="items">
            {#each visItems as item}
              <div class="item" class:item-done={item.completed}>
                <button class="item-move" onclick={() => moveRoutineItem(routine.id, item.id, 'up')} aria-label="Move item up" tabindex="-1"><ChevronUp size={7} strokeWidth={1.5} /></button>
                <button class="item-move" onclick={() => moveRoutineItem(routine.id, item.id, 'down')} aria-label="Move item down" tabindex="-1"><ChevronDown size={7} strokeWidth={1.5} /></button>
                <button class="item-check" class:checked={item.completed} onclick={() => toggleRoutineItem(routine.id, item.id)} aria-label={item.completed ? 'Uncheck' : 'Check'} role="checkbox" aria-checked={item.completed}>
                  {#if item.completed}<Check size={9} strokeWidth={2} />{/if}
                </button>
                <span class="item-title" class:item-title-done={item.completed}>{item.title}</span>
                {#if timeEdit[`${routine.id}-${item.id}`]}
                  <input class="te-input te-input-{CSS.escape(routine.id)}-{CSS.escape(item.id)}" type="number" min="0" max="999" placeholder="min" bind:value={timeInput[`${routine.id}-${item.id}`]} onblur={() => saveTime(routine.id, item.id)} onkeydown={(e) => handleTimeKey(e, routine.id, item.id)} aria-label="Time in minutes" />
                {:else}
                  <button class="item-time" class:active={item.timeEstimate} onclick={() => startTimeEdit(routine.id, item.id, item.timeEstimate)} aria-label={item.timeEstimate ? `Duration: ${formatTime(item.timeEstimate)}` : 'Set duration'}>
                    {#if item.timeEstimate}
                      <Clock size={8} strokeWidth={1.5} /> {formatTime(item.timeEstimate)}
                    {:else}
                      <Clock size={8} strokeWidth={1.5} />
                    {/if}
                  </button>
                {/if}
                <button class="item-del" onclick={() => removeRoutineItem(routine.id, item.id)} aria-label="Remove item">
                  <X size={9} strokeWidth={1.5} />
                </button>
              </div>
            {/each}
            {#if hasCompleted && routine.collapseCompleted}
              <button class="collapse-btn" onclick={() => toggleRoutineCollapse(routine.id)} aria-label="Show completed items">
                <ChevronRight size={10} strokeWidth={1.5} /> Show {routine.items.filter(i => i.completed).length} completed
              </button>
            {:else if hasCompleted}
              <button class="collapse-btn" onclick={() => toggleRoutineCollapse(routine.id)} aria-label="Hide completed items">
                <ChevronDown size={10} strokeWidth={1.5} /> Hide completed
              </button>
            {/if}
            <div class="add-item">
              <Plus size={11} strokeWidth={1.5} class="add-item-icon" />
              <input type="text" class="add-item-input" placeholder="Add item..." bind:value={itemInput[routine.id]} onkeydown={(e) => handleItemKey(e, routine.id)} aria-label="Add item" />
            </div>
          </div>
        </div>
      {/each}
    {/if}
  {/each}
</div>

{#if showCreate}
  <div class="dlg-overlay" role="dialog" onclick={() => showCreate = false} onkeydown={(e) => { if (e.key === 'Escape') showCreate = false }} tabindex="0">
    <div class="dlg" onclick={(e) => e.stopPropagation()} role="document">
      <div class="dlg-header">
        <span class="dlg-title">New {createType} routine</span>
        <button class="dlg-close" onclick={() => showCreate = false} aria-label="Close">
          <X size={15} strokeWidth={1.5} />
        </button>
      </div>
      <div class="dlg-body">
        <div class="dlg-field">
          <label class="dlg-label">Name</label>
          <input type="text" class="dlg-name-input" placeholder="Routine name" bind:value={createName} onkeydown={(e) => { if (e.key === 'Enter') confirmCreate() }} />
        </div>
        <div class="dlg-field">
          <label class="dlg-label">Duration <span class="dlg-opt">(optional)</span></label>
          <input type="number" class="dlg-name-input" min="0" max="999" placeholder="Minutes" bind:value={createTime} onkeydown={(e) => { if (e.key === 'Enter') confirmCreate() }} />
        </div>
      </div>
      <div class="dlg-footer">
        <button class="dlg-btn dlg-btn-secondary" onclick={() => showCreate = false}>Cancel</button>
        <button class="dlg-btn dlg-btn-primary" onclick={confirmCreate} disabled={!createName.trim()}>Create</button>
      </div>
    </div>
  </div>
{/if}

<style>
  /* ---- Section Headings ---- */
  .section-heading {
    font-size: 13px; font-weight: 650; color: var(--text-secondary);
    margin: 20px 0 6px; padding-bottom: 6px;
    display: flex; align-items: center; gap: 5px;
    letter-spacing: 0.3px; text-transform: uppercase;
    border-bottom: 1px solid var(--border-light);
  }
  .section-heading:first-of-type { margin-top: 0; }

  /* ---- Empty States ---- */
  .section-empty {
    display: flex; flex-direction: column; align-items: center;
    padding: 18px 16px 20px; text-align: center;
    border: 1px dashed var(--border); border-radius: var(--radius-md);
    background: var(--surface); margin-bottom: 4px;
  }
  .section-empty-icon { color: var(--text-muted); opacity: 0.25; margin-bottom: 6px; }
  .section-empty-text { font-size: 13px; color: var(--text-secondary); margin: 0 0 10px; font-weight: 420; }
  .section-empty-btn {
    display: inline-flex; align-items: center; gap: 4px;
    padding: 6px 12px; font-size: 12px; font-weight: 550;
    color: var(--accent); background: var(--accent-subtle);
    border: 1px solid transparent; border-radius: 7px;
    cursor: pointer; transition: all 0.15s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .section-empty-btn:hover { background: rgba(var(--accent-rgb), 0.16); border-color: var(--accent-subtle); }
  .section-empty-btn:active { transform: scale(0.97); }

  /* ---- Cards ---- */
  .card {
    background: var(--surface); border-radius: var(--radius-md);
    border: 1px solid var(--border); margin-bottom: 8px;
    padding: 12px 16px;
    transition: all 0.18s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .card:hover { box-shadow: 0 1px 6px rgba(0,0,0,0.04); border-color: var(--accent-subtle); }

  .card-header { display: flex; align-items: center; justify-content: space-between; gap: 4px; margin-bottom: 6px; }
  .card-header-left { display: flex; align-items: center; gap: 1px; min-width: 0; flex: 1; }
  .card-header-right { display: flex; align-items: center; gap: 1px; flex-shrink: 0; margin-left: 8px; }

  .card-move {
    width: 20px; height: 20px; border-radius: 4px;
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; color: var(--text-muted); background: transparent;
    border: none; padding: 0; opacity: 0;
    transition: opacity 0.12s cubic-bezier(0.34, 0.69, 0.4, 1), background 0.12s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .card:hover .card-move, .card:focus-within .card-move { opacity: 1; }
  .card-move:hover { color: var(--text); background: var(--surface-hover); }
  .card-move:focus-visible { opacity: 1; outline: 2px solid var(--accent); outline-offset: 1px; }

  .card-title {
    font-size: 15px; font-weight: 600; color: var(--text);
    white-space: nowrap; overflow: hidden; text-overflow: ellipsis;
  }

  /* ---- Badges ---- */
  .badge {
    display: inline-flex; align-items: center; gap: 2px;
    font-size: 9.5px; font-weight: 650;
    padding: 2px 6px; border-radius: 5px;
    margin-left: 6px; white-space: nowrap; flex-shrink: 0;
  }
  .badge-streak { color: #b8860b; background: rgba(184,134,11,0.08); }
  .badge-time { color: var(--text-muted); background: var(--surface); border: 1px solid var(--border-light); }

  .card-progress { font-size: 10.5px; font-weight: 550; color: var(--text-muted); margin-right: 2px; font-variant-numeric: tabular-nums; letter-spacing: -0.2px; }
  .card-act {
    width: 26px; height: 26px; border-radius: 5px;
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; color: var(--text-muted); background: transparent;
    border: none; padding: 0;
    transition: all 0.12s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .card-act:hover { color: var(--text); background: var(--surface-hover); }
  .card-act:active { transform: scale(0.92); }
  .card-act:focus-visible { outline: 2px solid var(--accent); outline-offset: 1px; }
  .card-act-del:hover { color: var(--danger); background: var(--danger-bg); }
  .card-act-del:focus-visible { outline-color: var(--danger); }

  /* ---- Progress Bar ---- */
  .progress-track { width: 100%; height: 3px; background: var(--border-light); border-radius: 2px; margin-bottom: 8px; overflow: hidden; }
  .progress-fill { height: 100%; background: var(--complete); border-radius: 2px; transition: width 0.35s cubic-bezier(0.34, 0.69, 0.4, 1); }

  /* ---- Items ---- */
  .items { display: flex; flex-direction: column; gap: 1px; }
  .item {
    display: flex; align-items: center; gap: 2px;
    padding: 2px 2px 2px 0; border-radius: 5px;
    transition: background 0.12s cubic-bezier(0.34, 0.69, 0.4, 1), opacity 0.35s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .item:hover { background: var(--surface-hover); }
  .item-done { opacity: 0.45; }
  .item-done:hover { opacity: 0.7; }

  .item-move {
    width: 16px; height: 18px; border-radius: 3px;
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; color: var(--text-muted); background: transparent;
    border: none; padding: 0; opacity: 0;
    transition: opacity 0.12s cubic-bezier(0.34, 0.69, 0.4, 1), background 0.12s cubic-bezier(0.34, 0.69, 0.4, 1);
    flex-shrink: 0;
  }
  .item:hover .item-move, .item:focus-within .item-move { opacity: 1; }
  .item-move:hover { color: var(--text); background: var(--surface-hover); }
  .item-move:active { transform: scale(0.9); }
  .item-move:focus-visible { opacity: 1; outline: 2px solid var(--accent); outline-offset: 1px; }

  .item-check {
    width: 17px; height: 17px; border-radius: 50%;
    border: 1.5px solid var(--border);
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; background: transparent;
    padding: 0; margin: 0 4px;
    transition: all 0.18s cubic-bezier(0.34, 1.56, 0.64, 1);
    flex-shrink: 0;
  }
  .item-check:hover { border-color: var(--complete); }
  .item-check:active { transform: scale(0.82); }
  .item-check.checked { background: var(--complete); border-color: var(--complete); color: #fff; }
  .item-check.checked:active { transform: scale(0.9); }
  .item-check:focus-visible { outline: 2px solid var(--accent); outline-offset: 2px; }

  .item-title { font-size: 13.5px; color: var(--text); flex: 1; min-width: 0; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; font-weight: 420; }
  .item-title-done { text-decoration: line-through; color: var(--text-muted); font-weight: 400; }

  .item-time {
    display: inline-flex; align-items: center; gap: 2px;
    padding: 1px 5px; border-radius: 4px;
    font-size: 10px; font-weight: 550;
    color: var(--text-muted); background: transparent;
    border: none; cursor: pointer; flex-shrink: 0;
    transition: all 0.12s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .item-time:hover { color: var(--text); background: var(--surface-hover); }
  .item-time.active { color: var(--accent); background: var(--accent-subtle); }
  .item-time:focus-visible { outline: 2px solid var(--accent); outline-offset: 1px; }

  .te-input {
    width: 40px; padding: 1px 3px; font-size: 10px;
    border-radius: 4px; border: 1px solid var(--accent);
    background: var(--bg); color: var(--text);
    text-align: center; outline: none; flex-shrink: 0;
    transition: box-shadow 0.12s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .te-input:focus { box-shadow: 0 0 0 2px rgba(var(--accent-rgb), 0.15); }

  .item-del {
    width: 22px; height: 22px; border-radius: 4px;
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; color: var(--text-muted); background: transparent;
    border: none; padding: 0; flex-shrink: 0;
    transition: all 0.12s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .item-del:hover { background: var(--danger-bg); color: var(--danger); }
  .item-del:active { transform: scale(0.9); }
  .item-del:focus-visible { outline: 2px solid var(--danger); outline-offset: 1px; }

  /* ---- Collapse ---- */
  .collapse-btn {
    display: flex; align-items: center; gap: 3px;
    padding: 4px 6px; margin-top: 1px;
    border-radius: 4px; font-size: 10.5px; font-weight: 550;
    color: var(--text-muted); background: transparent;
    border: none; cursor: pointer; width: fit-content;
    transition: all 0.12s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .collapse-btn:hover { color: var(--text); background: var(--surface-hover); }
  .collapse-btn:active { transform: scale(0.97); }
  .collapse-btn:focus-visible { outline: 2px solid var(--accent); outline-offset: 1px; }
  .collapse-btn svg { transition: transform 0.15s cubic-bezier(0.34, 0.69, 0.4, 1); }
  .collapse-btn:hover svg { transform: translateY(1px); }

  /* ---- Add Item ---- */
  .add-item { position: relative; padding: 2px 0 0; }
  .add-item-icon { position: absolute; left: 9px; top: 9px; color: var(--text-muted); pointer-events: none; opacity: 0.5; }
  .add-item-input {
    width: 100%; padding: 6px 10px 6px 26px;
    background: transparent; border: 1px solid transparent;
    border-radius: 5px; color: var(--text);
    font-size: 13px; font-weight: 420;
    outline: none; box-sizing: border-box;
    transition: border-color 0.15s cubic-bezier(0.34, 0.69, 0.4, 1), background 0.15s cubic-bezier(0.34, 0.69, 0.4, 1), box-shadow 0.15s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .add-item-input:hover { background: var(--surface-hover); }
  .add-item-input:focus { background: var(--surface); border-color: var(--accent); box-shadow: 0 0 0 2px rgba(var(--accent-rgb), 0.08); }
  .add-item-input::placeholder { color: var(--text-muted); font-weight: 400; }

  /* ---- Summary Bar ---- */
  .summary-bar {
    display: flex; gap: 18px; padding: 8px 14px;
    background: var(--surface); border: 1px solid var(--border);
    border-radius: var(--radius-sm); margin-bottom: 14px;
    font-size: 11.5px; color: var(--text-secondary); font-weight: 450;
  }
  .summary-item { display: flex; align-items: center; gap: 5px; }
  .summary-count { font-weight: 650; color: var(--text); font-variant-numeric: tabular-nums; }

  /* ---- Create Dialog ---- */
  .dlg-overlay {
    position: fixed; inset: 0; background: rgba(0,0,0,0.4);
    display: flex; align-items: center; justify-content: center;
    padding: 24px; z-index: 210;
    backdrop-filter: blur(16px); -webkit-backdrop-filter: blur(16px);
  }
  .dlg {
    background: var(--glass-bg); backdrop-filter: blur(var(--glass-blur));
    border: 1px solid var(--glass-border);
    border-radius: var(--radius-xl); width: 100%; max-width: 360px;
    box-shadow: var(--shadow-xl); overflow: hidden;
    animation: scaleIn 0.2s cubic-bezier(0.16, 1, 0.3, 1);
  }
  .dlg-header {
    display: flex; align-items: center; gap: 12px;
    padding: 16px 20px; border-bottom: 1px solid var(--glass-border);
  }
  .dlg-title { font-size: 15px; font-weight: 600; color: var(--text); flex: 1; }
  .dlg-close {
    width: 32px; height: 32px; border-radius: var(--radius-sm);
    display: flex; align-items: center; justify-content: center;
    cursor: pointer; color: var(--text-muted); background: transparent;
    border: none; transition: all 0.2s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .dlg-close:hover { background: var(--surface-hover); color: var(--text); transform: scale(1.05); }
  .dlg-body { padding: 20px; display: flex; flex-direction: column; gap: 16px; }
  .dlg-field { display: flex; flex-direction: column; gap: 6px; }
  .dlg-label {
    font-size: 12px; font-weight: 600; color: var(--text-secondary);
    text-transform: uppercase; letter-spacing: 0.4px;
  }
  .dlg-opt { font-weight: 400; color: var(--text-muted); text-transform: none; letter-spacing: 0; }
  .dlg-name-input {
    width: 100%; padding: 10px 14px; font-size: 14px;
    border-radius: var(--radius-md); border: 1px solid var(--border);
    background: var(--surface); color: var(--text);
    outline: none; box-sizing: border-box;
    transition: border-color 0.15s cubic-bezier(0.34, 0.69, 0.4, 1), box-shadow 0.15s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .dlg-name-input:focus { border-color: var(--accent); box-shadow: 0 0 0 3px rgba(var(--accent-rgb), 0.12); }
  .dlg-name-input::placeholder { color: var(--text-muted); }
  .dlg-footer {
    display: flex; gap: 10px; justify-content: flex-end;
    padding: 14px 20px; border-top: 1px solid var(--glass-border);
  }
  .dlg-btn {
    padding: 9px 20px; border-radius: var(--radius-md);
    font-size: 13px; font-weight: 500; cursor: pointer;
    transition: all 0.15s cubic-bezier(0.34, 0.69, 0.4, 1);
  }
  .dlg-btn-secondary { background: var(--surface-raised); color: var(--text-secondary); border: 1px solid var(--glass-border); }
  .dlg-btn-secondary:hover { background: var(--surface-hover); color: var(--text); }
  .dlg-btn-primary { background: var(--accent); color: #fff; border: none; }
  .dlg-btn-primary:hover { filter: brightness(1.08); }
  .dlg-btn-primary:active { transform: scale(0.97); }
  .dlg-btn-primary:disabled { opacity: 0.3; cursor: default; transform: none; filter: none; }

  @keyframes scaleIn { from { opacity: 0; transform: scale(0.95) translateY(8px); } to { opacity: 1; transform: scale(1) translateY(0); } }
</style>