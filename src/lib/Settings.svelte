<script>
  import ConfirmDialog from './ConfirmDialog.svelte'
  import SetPassword from './SetPassword.svelte'
  import { loadLocale, setLocale, getLocale, LANGUAGES } from './i18n.svelte.js'

  let locale = $state(getLocale())
  import { exportData, importData, loadPoints, computeStreak, requestNotificationPermission } from './taskStore.svelte.js'
  import { Palette, Bell, User, Database, Info, ChevronRight, ChevronLeft, Sun, Moon, Monitor, Globe, Clock, Lock, LogOut, Download, Upload, Trash2, RotateCcw, Star, Flame } from 'lucide-svelte'

  let { theme, effectiveTheme, onThemeCycle, accentColor, onAccentChange, autoThemeTime, onAutoThemeChange } = $props()

  let colorInput = $state(accentColor || '')
  let points = $state(loadPoints())
  let streak = $state(computeStreak())
  let ntfyEnabled = $state(localStorage.getItem('focus-ntfy-enabled') === 'true')
  let confirmAction = $state(null)
  let page = $state('main')

  const PRESET_COLORS = ['#5a9a9a', '#7c6db0', '#d4845a', '#5a8ab5', '#b06b8a', '#6aab6a', '#c4a55a', '#b06060']

  function toggleNtfy() {
    ntfyEnabled = !ntfyEnabled
    localStorage.setItem('focus-ntfy-enabled', ntfyEnabled ? 'true' : 'false')
    if (ntfyEnabled) requestNotificationPermission()
  }

  async function handleExport() {
    const json = await exportData()
    const blob = new Blob([json], { type: 'application/json' })
    const url = URL.createObjectURL(blob)
    const a = document.createElement('a')
    a.href = url; a.download = `focus-backup-${new Date().toISOString().split('T')[0]}.json`; a.click()
    URL.revokeObjectURL(url)
  }

  function handleImport() {
    const input = document.createElement('input')
    input.type = 'file'; input.accept = '.json'
    input.onchange = async () => {
      try {
        const text = await input.files[0].text()
        await importData(text)
        points = loadPoints(); streak = computeStreak()
      } catch (e) { alert('Invalid backup file') }
    }
    input.click()
  }

  function requestClearAll() { confirmAction = 'clear' }
  function requestSignOut() { confirmAction = 'signout' }
  function requestReset() { confirmAction = 'reset' }

  function doConfirm() {
    if (confirmAction === 'clear') {
      localStorage.clear()
      window.location.reload()
    } else if (confirmAction === 'signout') {
      localStorage.removeItem('focus-account-hash')
      localStorage.removeItem('focus-account-user')
      localStorage.removeItem('focus-session-expiry')
      localStorage.removeItem('focus-session-activity')
      localStorage.removeItem('focus-onboarded')
      window.location.reload()
    } else if (confirmAction === 'reset') {
      const tasks = JSON.parse(localStorage.getItem('focus-tasks') || '[]')
      for (const t of tasks) { t.rolloverCount = 0 }
      localStorage.setItem('focus-tasks', JSON.stringify(tasks))
      window.location.reload()
    }
    confirmAction = null
  }

  let themeLabel = $derived(theme === 'dark' ? 'Dark' : theme === 'light' ? 'Light' : 'System')

  let hasAccount = $state(!!localStorage.getItem('focus-account-hash'))
  let userName = $state(localStorage.getItem('focus-account-user') || '—')
</script>

<div class="sp-page">
  {#if page === 'main'}
    <div class="sp-header">
      <h1 class="sp-title">Settings</h1>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Appearance</span>
        <span class="sp-section-desc">Customize your visual experience</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row" onclick={() => page = 'appearance'}>
        <Palette size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Appearance</span>
          <span class="sp-row-value">{themeLabel} · {locale === 'en' ? 'English' : 'Español'}</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Notifications</span>
        <span class="sp-section-desc">Manage alerts and reminders</span>
      </div>
      <div class="sp-divider"></div>
      <div class="sp-row">
        <Bell size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Desktop Notifications</span>
          <span class="sp-row-value">Timed tasks & session alerts</span>
        </div>
        <button class="sp-toggle" class:sp-toggle--on={ntfyEnabled} onclick={toggleNtfy} role="switch" aria-checked={ntfyEnabled} aria-label="Desktop Notifications">
          <span class="sp-toggle-knob"></span>
        </button>
      </div>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Account</span>
        <span class="sp-section-desc">Manage your account and security</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row" onclick={() => page = 'account'}>
        <User size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Account</span>
          <span class="sp-row-value">{userName}</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Data</span>
        <span class="sp-section-desc">Backup, restore, and manage your data</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row" onclick={() => page = 'data'}>
        <Database size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Data Management</span>
          <span class="sp-row-value">Export, import, and storage</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Progress</span>
        <span class="sp-section-desc">Your achievements at a glance</span>
      </div>
      <div class="sp-divider"></div>
      <div class="sp-row sp-row--static">
        <Star size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Points</span>
          <span class="sp-row-value">{points} total</span>
        </div>
      </div>
      <div class="sp-row-inner-divider"></div>
      <div class="sp-row sp-row--static">
        <Flame size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Streak</span>
          <span class="sp-row-value">{streak} day{streak !== 1 ? 's' : ''}</span>
        </div>
      </div>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">About</span>
        <span class="sp-section-desc">App information</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row" onclick={() => page = 'about'}>
        <Info size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">About Sola</span>
          <span class="sp-row-value">Version 1.0</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

  {:else if page === 'appearance'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'main'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">Appearance</h1>
    </div>

    <div class="sp-section">
      <button class="sp-row" onclick={onThemeCycle}>
        {#if effectiveTheme === 'dark'}<Moon size={20} strokeWidth={1.5} class="sp-icon" />
        {:else if effectiveTheme === 'light'}<Sun size={20} strokeWidth={1.5} class="sp-icon" />
        {:else}<Monitor size={20} strokeWidth={1.5} class="sp-icon" />{/if}
        <div class="sp-row-body">
          <span class="sp-row-label">Theme</span>
          <span class="sp-row-value">{themeLabel}</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
      <div class="sp-row-inner-divider"></div>
      <button class="sp-row" onclick={() => page = 'accent'}>
        <Palette size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Accent Color</span>
          <span class="sp-row-value">{accentColor || 'Default'}</span>
        </div>
        <div class="sp-row-end">
          {#if accentColor}
            <span class="sp-dot" style="background:{accentColor}"></span>
          {/if}
          <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
        </div>
      </button>
      <div class="sp-row-inner-divider"></div>
      <button class="sp-row" onclick={() => page = 'auto-theme'}>
        <Clock size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Auto Theme</span>
          <span class="sp-row-value">{autoThemeTime || 'Off'}</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
      <div class="sp-row-inner-divider"></div>
      <button class="sp-row" onclick={() => page = 'language'}>
        <Globe size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Language</span>
          <span class="sp-row-value">{locale === 'en' ? 'English' : 'Español'}</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

  {:else if page === 'accent'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'appearance'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">Accent Color</h1>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Preset Colors</span>
      </div>
      <div class="sp-divider"></div>
      <div class="sp-color-grid">
        {#each PRESET_COLORS as c}
          <button class="sp-swatch" class:sp-swatch--active={accentColor === c} style="background:{c}" onclick={() => { colorInput = c; onAccentChange(c) }} aria-label={c}>
            {#if accentColor === c}<span class="sp-swatch-check">✓</span>{/if}
          </button>
        {/each}
      </div>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Custom Color</span>
      </div>
      <div class="sp-divider"></div>
      <div class="sp-custom-row">
        <input type="color" class="sp-picker" value={accentColor || '#5a9a9a'} oninput={(e) => { const v = e.target.value; colorInput = v; onAccentChange(v) }} />
        <div class="sp-hex-wrap">
          <span class="sp-hex-hash">#</span>
          <input type="text" class="sp-hex-input" placeholder="5a9a9a" value={colorInput.replace('#', '')} oninput={(e) => { const v = '#' + e.target.value.replace('#', ''); if (/^#[0-9a-fA-F]{6}$/.test(v)) { colorInput = v; onAccentChange(v) } }} />
        </div>
        {#if accentColor}
          <button class="sp-reset-btn" onclick={() => { colorInput = ''; onAccentChange('') }} aria-label="Reset accent">
            <RotateCcw size={16} strokeWidth={1.5} />
          </button>
        {/if}
      </div>
    </div>

  {:else if page === 'auto-theme'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'appearance'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">Auto Theme</h1>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Schedule</span>
        <span class="sp-section-desc">Automatically switch between light and dark mode</span>
      </div>
      <div class="sp-divider"></div>
      <div class="sp-auto-row">
        <input type="time" class="sp-time-input" value={autoThemeTime} oninput={(e) => { const v = e.target.value; localStorage.setItem('focus-auto-theme-time', v); onAutoThemeChange(v) }} />
        {#if autoThemeTime}
          <button class="sp-clear-btn" onclick={() => { localStorage.setItem('focus-auto-theme-time', ''); onAutoThemeChange('') }}>Clear</button>
        {/if}
      </div>
    </div>

  {:else if page === 'language'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'appearance'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">Language</h1>
    </div>

    <div class="sp-section">
      {#each LANGUAGES as lang, i}
        {#if i > 0}<div class="sp-row-inner-divider"></div>{/if}
        <button class="sp-row" class:sp-row--selected={locale === lang.code} onclick={() => { setLocale(lang.code); locale = lang.code }}>
          <Globe size={20} strokeWidth={1.5} class="sp-icon" />
          <div class="sp-row-body">
            <span class="sp-row-label">{lang.label}</span>
          </div>
          {#if locale === lang.code}
            <span class="sp-check">✓</span>
          {/if}
        </button>
      {/each}
    </div>

  {:else if page === 'notifications'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'main'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">Notifications</h1>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">General</span>
      </div>
      <div class="sp-divider"></div>
      <div class="sp-row">
        <Bell size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Desktop Notifications</span>
          <span class="sp-row-value">Alerts for tasks and sessions</span>
        </div>
        <button class="sp-toggle" class:sp-toggle--on={ntfyEnabled} onclick={toggleNtfy} role="switch" aria-checked={ntfyEnabled} aria-label="Desktop Notifications">
          <span class="sp-toggle-knob"></span>
        </button>
      </div>
    </div>

  {:else if page === 'account'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'main'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">Account</h1>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Profile</span>
      </div>
      <div class="sp-divider"></div>
      <div class="sp-row sp-row--static">
        <User size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Signed in as</span>
          <span class="sp-row-value">{userName}</span>
        </div>
      </div>
      {#if !hasAccount}
        <SetPassword onSet={() => { hasAccount = true; userName = localStorage.getItem('focus-account-user') || 'User' }} />
      {:else}
        <div class="sp-row-inner-divider"></div>
        <div class="sp-row sp-row--static">
          <Lock size={20} strokeWidth={1.5} class="sp-icon" />
          <div class="sp-row-body">
            <span class="sp-row-label">Password</span>
            <span class="sp-row-value">••••••</span>
          </div>
        </div>
      {/if}
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Actions</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row sp-row--danger" onclick={requestSignOut}>
        <LogOut size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label sp-red">Sign Out</span>
          <span class="sp-row-value">Sign out of your account</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

  {:else if page === 'data'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'main'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">Data</h1>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Backup</span>
        <span class="sp-section-desc">Export or import your data</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row" onclick={handleExport}>
        <Download size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Export Data</span>
          <span class="sp-row-value">Save a backup file</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
      <div class="sp-row-inner-divider"></div>
      <button class="sp-row" onclick={handleImport}>
        <Upload size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Import Data</span>
          <span class="sp-row-value">Restore from a backup</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Storage</span>
        <span class="sp-section-desc">Dangerous actions</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row sp-row--danger" onclick={requestClearAll}>
        <Trash2 size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label sp-red">Clear All Data</span>
          <span class="sp-row-value">This cannot be undone</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

    <div class="sp-section">
      <div class="sp-section-header">
        <span class="sp-section-title">Maintenance</span>
      </div>
      <div class="sp-divider"></div>
      <button class="sp-row" onclick={requestReset}>
        <RotateCcw size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Reset Without Guilt</span>
          <span class="sp-row-value">Clear rollover counters</span>
        </div>
        <ChevronRight size={16} strokeWidth={1.5} class="sp-chevron" />
      </button>
    </div>

  {:else if page === 'about'}
    <div class="sp-header">
      <button class="sp-back" onclick={() => page = 'main'}>
        <ChevronLeft size={22} strokeWidth={1.5} />
      </button>
      <h1 class="sp-title">About</h1>
    </div>

    <div class="sp-about">
      <div class="sp-about-logo">Sola</div>
      <div class="sp-about-version">Version 1.0</div>
      <div class="sp-about-desc">A focus-friendly productivity app designed for ADHD minds.</div>
    </div>

    <div class="sp-section">
      <div class="sp-row sp-row--static">
        <Star size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Points</span>
          <span class="sp-row-value">{points} total</span>
        </div>
      </div>
      <div class="sp-row-inner-divider"></div>
      <div class="sp-row sp-row--static">
        <Flame size={20} strokeWidth={1.5} class="sp-icon" />
        <div class="sp-row-body">
          <span class="sp-row-label">Streak</span>
          <span class="sp-row-value">{streak} day{streak !== 1 ? 's' : ''}</span>
        </div>
      </div>
    </div>
  {/if}
</div>

<ConfirmDialog open={confirmAction === 'clear'} title="Clear all data" message="Delete all data? This cannot be undone." confirmLabel="Delete all" onConfirm={doConfirm} onCancel={() => confirmAction = null} />
<ConfirmDialog open={confirmAction === 'signout'} title="Sign out" message="Sign out? You will need your password to sign back in." confirmLabel="Sign out" onConfirm={doConfirm} onCancel={() => confirmAction = null} />
<ConfirmDialog open={confirmAction === 'reset'} title="Reset without guilt" message="This will set all rollover counters to zero and give you a fresh start. Your tasks, points, streaks, and all other data will be preserved." confirmLabel="Reset counters" onConfirm={doConfirm} onCancel={() => confirmAction = null} />

<style>
  .sp-page {
    display: flex;
    flex-direction: column;
    height: 100%;
  }
  .sp-header {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 24px 24px 20px;
    flex-shrink: 0;
  }
  .sp-back {
    width: 36px; height: 36px;
    display: flex; align-items: center; justify-content: center;
    background: transparent; border: none;
    color: var(--accent);
    cursor: pointer;
    border-radius: 8px;
    transition: background 0.12s;
    flex-shrink: 0;
  }
  .sp-back:hover { background: var(--surface-hover); }
  .sp-title {
    flex: 1;
    font-size: 24px;
    font-weight: 650;
    letter-spacing: -0.5px;
    color: var(--text);
    margin: 0;
  }
  .sp-section {
    padding: 0 24px;
    margin-bottom: 32px;
  }
  .sp-section-header {
    padding: 0 0 8px;
  }
  .sp-section-title {
    font-size: 15px;
    font-weight: 600;
    color: var(--text);
    display: block;
  }
  .sp-section-desc {
    font-size: 12px;
    color: var(--text-muted);
    display: block;
    margin-top: 2px;
  }
  .sp-divider {
    height: 1px;
    background: var(--border);
    margin-bottom: 4px;
  }
  .sp-row {
    display: flex;
    align-items: center;
    gap: 14px;
    width: 100%;
    padding: 14px 0;
    border: none;
    background: transparent;
    color: var(--text);
    cursor: pointer;
    text-align: left;
    transition: opacity 0.12s;
  }
  .sp-row:hover { opacity: 0.7; }
  .sp-row--static { cursor: default; }
  .sp-row--static:hover { opacity: 1; }
  .sp-row--danger:hover { opacity: 0.7; }
  .sp-row--selected { opacity: 0.7; }
  :global(.sp-icon) {
    color: var(--text-secondary);
    flex-shrink: 0;
  }
  .sp-row-body {
    flex: 1;
    min-width: 0;
  }
  .sp-row-label {
    font-size: 15px;
    font-weight: 400;
    color: var(--text);
    display: block;
    line-height: 1.4;
  }
  .sp-row-value {
    font-size: 12px;
    color: var(--text-muted);
    display: block;
    margin-top: 1px;
    line-height: 1.3;
  }
  .sp-red { color: #b06060; }
  .sp-row-end {
    display: flex;
    align-items: center;
    gap: 6px;
  }
  :global(.sp-chevron) {
    color: var(--text-muted);
    flex-shrink: 0;
    opacity: 0.4;
  }
  .sp-dot {
    width: 10px; height: 10px;
    border-radius: 50%;
    flex-shrink: 0;
  }
  .sp-check {
    font-size: 16px;
    color: var(--accent);
    font-weight: 700;
    flex-shrink: 0;
  }
  .sp-row-inner-divider {
    height: 0.5px;
    background: var(--border);
    opacity: 0.5;
  }
  .sp-toggle {
    width: 44px; height: 24px;
    border-radius: 12px;
    background: var(--border);
    border: none;
    cursor: pointer;
    position: relative;
    transition: background 0.25s;
    flex-shrink: 0;
    padding: 0;
  }
  .sp-toggle--on { background: var(--accent); }
  .sp-toggle-knob {
    width: 18px; height: 18px;
    border-radius: 50%;
    background: #fff;
    position: absolute;
    top: 3px; left: 3px;
    transition: left 0.25s;
    box-shadow: 0 1px 3px rgba(0,0,0,0.2);
  }
  .sp-toggle--on .sp-toggle-knob { left: 23px; }

  .sp-color-grid {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 12px;
    padding: 16px 0 8px;
  }
  .sp-swatch {
    aspect-ratio: 1;
    border-radius: 50%;
    border: none;
    cursor: pointer;
    display: flex; align-items: center; justify-content: center;
    transition: transform 0.15s;
    padding: 0;
    position: relative;
  }
  .sp-swatch:hover { transform: scale(1.1); }
  .sp-swatch--active {
    box-shadow: 0 0 0 2px var(--bg), 0 0 0 4px currentColor;
  }
  .sp-swatch-check {
    color: #fff;
    font-size: 16px;
    font-weight: 700;
  }

  .sp-custom-row {
    display: flex;
    align-items: center;
    gap: 12px;
    padding: 12px 0 8px;
  }
  .sp-picker {
    width: 40px; height: 40px;
    border: none;
    border-radius: 50%;
    cursor: pointer;
    background: none;
    padding: 0;
    flex-shrink: 0;
  }
  .sp-picker::-webkit-color-swatch-wrapper { padding: 0; }
  .sp-picker::-webkit-color-swatch { border-radius: 50%; }
  .sp-hex-wrap {
    flex: 1;
    display: flex;
    align-items: center;
    border-bottom: 1px solid var(--border);
    padding: 0 0 4px;
  }
  .sp-hex-hash {
    font-size: 15px;
    color: var(--text-muted);
    font-family: var(--font-mono);
  }
  .sp-hex-input {
    flex: 1;
    border: none;
    background: transparent;
    color: var(--text);
    font-size: 15px;
    font-family: var(--font-mono);
    padding: 8px 0;
    outline: none;
  }
  .sp-reset-btn {
    width: 36px; height: 36px;
    display: flex; align-items: center; justify-content: center;
    background: transparent;
    border: none;
    color: var(--text-muted);
    cursor: pointer;
    border-radius: 8px;
    transition: all 0.12s;
    flex-shrink: 0;
  }
  .sp-reset-btn:hover { color: var(--danger); background: rgba(176,96,96,0.1); }

  .sp-auto-row {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 12px 0 8px;
  }
  .sp-time-input {
    padding: 10px 0;
    border: none;
    border-bottom: 1px solid var(--border);
    background: transparent;
    color: var(--text);
    font-size: 15px;
    transition: border-color 0.2s;
    flex: 1;
    outline: none;
  }
  .sp-time-input:focus { border-color: var(--accent); }
  .sp-clear-btn {
    padding: 8px 14px;
    border-radius: 6px;
    border: none;
    background: transparent;
    color: var(--text-secondary);
    font-size: 13px;
    font-weight: 500;
    cursor: pointer;
    transition: all 0.12s;
  }
  .sp-clear-btn:hover { color: var(--danger); background: rgba(176,96,96,0.08); }

  .sp-about {
    display: flex;
    flex-direction: column;
    align-items: center;
    padding: 48px 24px 32px;
    gap: 4px;
  }
  .sp-about-logo {
    font-size: 36px;
    font-weight: 700;
    letter-spacing: -1px;
    color: var(--accent);
  }
  .sp-about-version {
    font-size: 14px;
    color: var(--text-secondary);
    font-weight: 500;
  }
  .sp-about-desc {
    font-size: 13px;
    color: var(--text-muted);
    text-align: center;
    margin-top: 8px;
    line-height: 1.5;
    max-width: 280px;
  }
</style>
