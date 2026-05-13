<!-- src/lib/components/integrations/ConnectIntegrationDialog.svelte -->
<script lang="ts">
  import { integrationsStore } from '$lib/stores/integrations.svelte';

  interface Props {
    open: boolean;
    integrationName: string;
    integrationSlug: string;
    tokenLabel: string;
    tokenHint?: string;
    onClose: () => void;
  }

  let { open, integrationName, integrationSlug, tokenLabel, tokenHint, onClose }: Props = $props();

  let token = $state('');
  let isSubmitting = $state(false);
  let errorMessage = $state<string | null>(null);

  async function handleSubmit(e: SubmitEvent) {
    e.preventDefault();
    if (!token.trim()) {
      errorMessage = `${tokenLabel} is required`;
      return;
    }
    isSubmitting = true;
    errorMessage = null;
    try {
      await integrationsStore.connect(integrationSlug, {
        api_key: token.trim(),
        api_key_set: true,
      });
      if (integrationsStore.error) {
        errorMessage = integrationsStore.error;
        return;
      }
      token = '';
      onClose();
    } catch (err) {
      errorMessage = (err as Error)?.message || 'Failed to connect integration';
    } finally {
      isSubmitting = false;
    }
  }

  function handleBackdrop(e: MouseEvent) {
    if ((e.target as HTMLElement).classList.contains('cid-overlay')) onClose();
  }

  function handleKeyDown(e: KeyboardEvent) {
    if (e.key === 'Escape') onClose();
  }
</script>

{#if open}
  <!-- svelte-ignore a11y_click_events_have_key_events -->
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div
    class="cid-overlay"
    onclick={handleBackdrop}
    onkeydown={handleKeyDown}
    role="dialog"
    aria-modal="true"
    aria-label="Connect {integrationName}"
    tabindex="-1"
  >
    <div class="cid-modal">
      <header class="cid-header">
        <h2 class="cid-title">Connect {integrationName}</h2>
        <button class="cid-close" onclick={onClose} aria-label="Close dialog" type="button">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
            <path d="M18 6 6 18M6 6l12 12" />
          </svg>
        </button>
      </header>

      <form class="cid-form" onsubmit={handleSubmit} novalidate>
        <div class="cid-body">
          <label class="cid-label" for="cid-token">{tokenLabel}</label>
          <input
            id="cid-token"
            type="password"
            class="cid-input"
            bind:value={token}
            autocomplete="off"
            spellcheck="false"
            disabled={isSubmitting}
            placeholder="Paste your {tokenLabel.toLowerCase()}"
          />
          {#if tokenHint}
            <p class="cid-hint">{tokenHint}</p>
          {/if}
          {#if errorMessage}
            <div class="cid-error" role="alert">{errorMessage}</div>
          {/if}
        </div>

        <footer class="cid-footer">
          <button
            type="button"
            class="cid-btn cid-btn--secondary"
            onclick={onClose}
            disabled={isSubmitting}
          >
            Cancel
          </button>
          <button
            type="submit"
            class="cid-btn cid-btn--primary"
            disabled={isSubmitting}
            aria-busy={isSubmitting}
          >
            {#if isSubmitting}
              <span class="cid-spinner" aria-hidden="true"></span>
              Connecting…
            {:else}
              Connect
            {/if}
          </button>
        </footer>
      </form>
    </div>
  </div>
{/if}

<style>
  .cid-overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.6);
    backdrop-filter: blur(4px);
    -webkit-backdrop-filter: blur(4px);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
    padding: 24px;
  }

  .cid-modal {
    background: var(--bg-tertiary);
    border: 1px solid var(--border-default);
    border-radius: var(--radius-xl);
    width: 100%;
    max-width: 480px;
    display: flex;
    flex-direction: column;
    box-shadow: 0 24px 80px rgba(0, 0, 0, 0.5);
    overflow: hidden;
  }

  .cid-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 18px 20px;
    border-bottom: 1px solid var(--border-default);
  }

  .cid-title {
    font-size: 15px;
    font-weight: 600;
    color: var(--text-primary);
    margin: 0;
  }

  .cid-close {
    width: 28px;
    height: 28px;
    border-radius: var(--radius-xs);
    border: 1px solid transparent;
    background: transparent;
    color: var(--text-tertiary);
    cursor: pointer;
    display: flex;
    align-items: center;
    justify-content: center;
    transition: all 120ms ease;
  }

  .cid-close:hover {
    background: var(--bg-elevated);
    border-color: var(--border-default);
    color: var(--text-primary);
  }

  .cid-form {
    display: flex;
    flex-direction: column;
  }

  .cid-body {
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 8px;
  }

  .cid-label {
    font-size: 12px;
    font-weight: 500;
    color: var(--text-secondary);
  }

  .cid-input {
    height: 34px;
    padding: 0 10px;
    border: 1px solid var(--border-default);
    border-radius: var(--radius-sm);
    background: var(--bg-surface);
    color: var(--text-primary);
    font-size: 13px;
    font-family: var(--font-mono);
    transition: border-color 120ms ease;
  }

  .cid-input:focus {
    outline: none;
    border-color: var(--accent-primary);
  }

  .cid-input:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .cid-hint {
    font-size: 11px;
    color: var(--text-tertiary);
    margin: 2px 0 0;
    line-height: 1.4;
  }

  .cid-error {
    margin-top: 4px;
    padding: 8px 10px;
    border-radius: var(--radius-sm);
    background: rgba(239, 68, 68, 0.1);
    border: 1px solid rgba(239, 68, 68, 0.3);
    color: #fca5a5;
    font-size: 12px;
  }

  .cid-footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 14px 20px;
    border-top: 1px solid var(--border-default);
    background: var(--bg-secondary);
  }

  .cid-btn {
    height: 34px;
    padding: 0 16px;
    border-radius: var(--radius-sm);
    font-size: 13px;
    font-weight: 500;
    font-family: var(--font-sans);
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 6px;
    transition: all 120ms ease;
    border: 1px solid transparent;
  }

  .cid-btn--secondary {
    background: transparent;
    border-color: var(--border-default);
    color: var(--text-secondary);
  }

  .cid-btn--secondary:hover:not(:disabled) {
    background: var(--bg-elevated);
    border-color: var(--border-hover);
    color: var(--text-primary);
  }

  .cid-btn--primary {
    background: rgba(59, 130, 246, 0.2);
    border-color: rgba(59, 130, 246, 0.5);
    color: #93c5fd;
  }

  .cid-btn--primary:hover:not(:disabled) {
    background: rgba(59, 130, 246, 0.3);
    border-color: rgba(59, 130, 246, 0.7);
    color: #bfdbfe;
  }

  .cid-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .cid-spinner {
    width: 12px;
    height: 12px;
    border: 2px solid rgba(147, 197, 253, 0.3);
    border-top-color: #93c5fd;
    border-radius: 50%;
    animation: cid-spin 0.7s linear infinite;
  }

  @keyframes cid-spin {
    to { transform: rotate(360deg); }
  }
</style>
