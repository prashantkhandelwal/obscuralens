<script lang="ts">
  import {
    Check,
    Clipboard,
    Download,
    ExternalLink,
    ScanSearch,
    RotateCcw,
    ShieldCheck,
    SlidersHorizontal,
    Sparkles,
  } from 'lucide-svelte'
  import DOMPurify from 'dompurify'
  import { marked } from 'marked'

  type AnalysisMode = 'summary' | 'links' | 'assets' | 'screenshot'

  const analysisModes: { value: AnalysisMode; label: string; resultLabel: string }[] = [
    { value: 'summary', label: 'Page summary', resultLabel: 'Summary' },
    { value: 'links', label: 'Links', resultLabel: 'Link map' },
    { value: 'assets', label: 'Assets', resultLabel: 'Asset inventory' },
    { value: 'screenshot', label: 'Screenshot', resultLabel: 'Page screenshot' },
  ]

  let url = ''
  let mode: AnalysisMode = 'summary'
  let resultMode: AnalysisMode = 'summary'
  let model = 'qwen3:0.6b'
  let maxChars = 30000
  let summary = ''
  let imageBase64 = ''
  let imageMimeType = 'image/png'
  let error = ''
  let loading = false
  let copied = false
  let showSettings = false

  $: renderedSummary = DOMPurify.sanitize(
    marked.parse(summary, { async: false }),
  )
  $: imageUrl = imageBase64 ? `data:${imageMimeType};base64,${imageBase64}` : ''

  async function submit(event: SubmitEvent) {
    event.preventDefault()
    const requestedMode = mode
    loading = true
    error = ''
    summary = ''
    imageBase64 = ''
    copied = false

    try {
      const response = await fetch('/api/summarize', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({
          url,
          mode,
          model: model.trim() || null,
          max_chars: maxChars,
        }),
      })
      const body = await response.json()
      if (!response.ok) {
        throw new Error(body.detail ?? 'The page could not be summarized.')
      }
      summary = body.summary ?? ''
      imageBase64 = body.image_base64 ?? ''
      imageMimeType = body.mime_type ?? 'image/png'
      resultMode = requestedMode
    } catch (caught) {
      error = caught instanceof Error ? caught.message : 'Something went wrong.'
    } finally {
      loading = false
    }
  }

  async function copySummary() {
    await navigator.clipboard.writeText(summary)
    copied = true
    window.setTimeout(() => (copied = false), 1800)
  }

  function reset() {
    url = ''
    summary = ''
    imageBase64 = ''
    error = ''
    copied = false
  }

  $: selectedMode = analysisModes.find((item) => item.value === mode) ?? analysisModes[0]
  $: completedMode = analysisModes.find((item) => item.value === resultMode) ?? analysisModes[0]
</script>

<svelte:head>
  <title>ObscuraLens | Local webpage summaries</title>
</svelte:head>

<header class="topbar">
  <a class="brand" href="/" aria-label="ObscuraLens home">
    <span class="brand-mark" aria-hidden="true"><ScanSearch size={20} /></span>
    <span>ObscuraLens</span>
  </a>
  <div class="header-meta">
    <span class="privacy"><ShieldCheck size={14} /> Private by design</span>
    <div class="local-status"><span></span> Agent online</div>
  </div>
</header>

<main>
  <section class="workspace" aria-labelledby="workspace-title">
    <div class="intro">
      <p class="eyebrow"><span></span> LOCAL RESEARCH AGENT</p>
      <h1 id="workspace-title">See the signal in every page.</h1>
      <p>ObscuraLens turns any public webpage into a focused brief, processed locally with your own AI stack.</p>
      <div class="stack-line" aria-label="Powered by Obscura, LangChain, and Ollama">
        <span>Obscura</span><i></i><span>LangChain</span><i></i><span>Ollama</span>
      </div>
    </div>

    <form class="composer" onsubmit={submit}>
      <div class="form-heading">
        <div>
          <span class="step">01</span>
          <label for="url">Paste a webpage</label>
        </div>
        <span class="format-note">PUBLIC HTTP(S) URL</span>
      </div>
      <div class="mode-field">
        <label for="mode">Choose what to inspect</label>
        <select id="mode" bind:value={mode} disabled={loading}>
          {#each analysisModes as option}
            <option value={option.value}>{option.label}</option>
          {/each}
        </select>
      </div>
      <div class="url-row">
        <div class="url-field">
          <ExternalLink size={19} aria-hidden="true" />
          <input
            id="url"
            type="url"
            bind:value={url}
            placeholder="https://example.com/article"
            required
            disabled={loading}
            autocomplete="url"
          />
        </div>
        <button class="primary" type="submit" disabled={loading || !url}>
          <Sparkles size={18} aria-hidden="true" />
          {loading ? 'Inspecting page...' : `Run ${selectedMode.label}`}
        </button>
      </div>

      <button
        class="settings-toggle"
        type="button"
        aria-expanded={showSettings}
        onclick={() => (showSettings = !showSettings)}
      >
        <SlidersHorizontal size={16} aria-hidden="true" />
        {showSettings ? 'Hide settings' : 'Tune agent'}
      </button>

      {#if showSettings}
        <div class="settings">
          <div>
            <label for="model">Ollama model</label>
            <input id="model" bind:value={model} placeholder="qwen3:0.6b" />
          </div>
          <div>
            <label for="max-chars">Snapshot limit</label>
            <input id="max-chars" type="number" bind:value={maxChars} min="1000" max="100000" step="1000" />
          </div>
        </div>
      {/if}
    </form>

    {#if error}
      <div class="notice error" role="alert">
        <strong>Could not create the brief.</strong>
        <span>{error}</span>
      </div>
    {/if}

    {#if loading}
      <div class="result loading-result" aria-live="polite">
        <div class="result-heading">
          <span class="pulse"></span>
          <span>Agent at work</span>
        </div>
        <div class="skeleton wide"></div>
        <div class="skeleton"></div>
        <div class="skeleton short"></div>
      </div>
    {:else if summary || imageBase64}
      <article class="result" aria-live="polite">
        <div class="result-heading">
          <div>
            <p class="eyebrow">YOUR BRIEF</p>
            <h2>{completedMode.resultLabel}</h2>
          </div>
          <div class="result-actions">
            {#if summary}
              <button class="icon-button" type="button" onclick={copySummary} title="Copy result" aria-label="Copy result">
                {#if copied}<Check size={18} />{:else}<Clipboard size={18} />{/if}
              </button>
            {:else}
              <a class="icon-button" href={imageUrl} download="obscuralens-screenshot.png" title="Download screenshot" aria-label="Download screenshot">
                <Download size={18} />
              </a>
            {/if}
            <button class="icon-button" type="button" onclick={reset} title="Start over" aria-label="Start over">
              <RotateCcw size={18} />
            </button>
          </div>
        </div>
        {#if imageBase64}
          <div class="screenshot-frame">
            <img src={imageUrl} alt={`Screenshot of ${url}`} />
          </div>
        {:else}
          <div class="summary-text">{@html renderedSummary}</div>
        {/if}
        <a class="source-link" href={url} target="_blank" rel="noreferrer">
          <ExternalLink size={15} aria-hidden="true" /> View source
        </a>
      </article>
    {:else}
      <div class="empty-state">
        <div class="empty-icon"><ScanSearch size={22} /></div>
        <div>
          <strong>Ready for a closer look</strong>
          <p>Your report or screenshot will appear here.</p>
        </div>
      </div>
    {/if}
  </section>
</main>

<footer>
  <span>ObscuraLens</span><span>Local-first webpage intelligence</span>
</footer>