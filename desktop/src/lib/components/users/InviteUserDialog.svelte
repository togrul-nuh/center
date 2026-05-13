<!-- src/lib/components/users/InviteUserDialog.svelte -->
<script lang="ts">
  import { invitations, type InvitationRecord } from '$api/client';
  import { organizationsStore } from '$lib/stores/organizations.svelte';
  import { toastStore } from '$lib/stores/toasts.svelte';

  interface Props {
    open: boolean;
    onClose: () => void;
  }

  let { open, onClose }: Props = $props();

  let email = $state('');
  let role = $state<'admin' | 'member' | 'viewer'>('member');
  let isSubmitting = $state(false);
  let errorMessage = $state<string | null>(null);
  let created = $state<InvitationRecord | null>(null);
  let copied = $state(false);

  function validateEmail(v: string): boolean {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(v);
  }

  async function handleSubmit(e: SubmitEvent) {
    e.preventDefault();
    if (!email.trim()) {
      errorMessage = 'Email is required';
      return;
    }
    if (!validateEmail(email.trim())) {
      errorMessage = 'Please enter a valid email address';
      return;
    }

    errorMessage = null;
    isSubmitting = true;
    try {
      const body: { email: string; role: string; organization_id?: string } = {
        email: email.trim(),
        role,
      };
      if (organizationsStore.current?.id) {
        body.organization_id = organizationsStore.current.id;
      }
      const res = await invitations.create(body);
      created = res.invitation;
      toastStore.success('Invitation created', email.trim());
    } catch (err) {
      errorMessage = (err as Error)?.message || 'Failed to create invitation';
    } finally {
      isSubmitting = false;
    }
  }

  async function copyToken() {
    if (!created) return;
    try {
      await navigator.clipboard.writeText(created.token);
      copied = true;
      setTimeout(() => (copied = false), 2000);
    } catch {
      // Clipboard write can fail in some Tauri sandboxes — fall back silently.
    }
  }

  function reset() {
    email = '';
    role = 'member';
    errorMessage = null;
    created = null;
    copied = false;
  }

  function handleClose() {
    reset();
    onClose();
  }

  function handleBackdrop(e: MouseEvent) {
    if ((e.target as HTMLElement).classList.contains('iud-overlay')) handleClose();
  }

  function handleKeyDown(e: KeyboardEvent) {
    if (e.key === 'Escape') handleClose();
  }
</script>

{#if open}
  <!-- svelte-ignore a11y_click_events_have_key_events -->
  <!-- svelte-ignore a11y_no_static_element_interactions -->
  <div
    class="iud-overlay"
    onclick={handleBackdrop}
    onkeydown={handleKeyDown}
    role="dialog"
    aria-modal="true"
    aria-label="Invite a user"
    tabindex="-1"
  >
    <div class="iud-modal">
      <header class="iud-header">
        <h2 class="iud-title">Invite User</h2>
        <button class="iud-close" onclick={handleClose} aria-label="Close dialog" type="button">
          <svg width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" aria-hidden="true">
            <path d="M18 6 6 18M6 6l12 12" />
          </svg>
        </button>
      </header>

      {#if created}
        <!-- Success state — show the token + copy button -->
        <div class="iud-body">
          <p class="iud-success-line">
            Invitation created for <strong>{created.email}</strong>. Share this token with them — it's valid for 7 days.
          </p>
          <label class="iud-label" for="iud-token">Invitation token</label>
          <div class="iud-token-row">
            <input
              id="iud-token"
              type="text"
              class="iud-input iud-input--mono"
              value={created.token}
              readonly
              onclick={(e) => (e.target as HTMLInputElement).select()}
            />
            <button class="iud-btn iud-btn--secondary" type="button" onclick={copyToken}>
              {copied ? 'Copied!' : 'Copy'}
            </button>
          </div>
          <p class="iud-hint">
            They can accept by calling <code>POST /api/v1/invitations/{created.token}/accept</code> while authenticated, or by visiting a future invite-acceptance page in Canopy.
          </p>
        </div>
        <footer class="iud-footer">
          <button type="button" class="iud-btn iud-btn--secondary" onclick={reset}>
            Invite another
          </button>
          <button type="button" class="iud-btn iud-btn--primary" onclick={handleClose}>
            Done
          </button>
        </footer>
      {:else}
        <!-- Form state -->
        <form class="iud-form" onsubmit={handleSubmit} novalidate>
          <div class="iud-body">
            <label class="iud-label" for="iud-email">Email</label>
            <input
              id="iud-email"
              type="email"
              class="iud-input"
              bind:value={email}
              placeholder="teammate@example.com"
              autocomplete="off"
              spellcheck="false"
              disabled={isSubmitting}
              required
            />

            <label class="iud-label" for="iud-role">Role</label>
            <select
              id="iud-role"
              class="iud-input"
              bind:value={role}
              disabled={isSubmitting}
            >
              <option value="admin">Admin — full access</option>
              <option value="member">Member — standard access</option>
              <option value="viewer">Viewer — read-only</option>
            </select>

            {#if errorMessage}
              <div class="iud-error" role="alert">{errorMessage}</div>
            {/if}
          </div>

          <footer class="iud-footer">
            <button
              type="button"
              class="iud-btn iud-btn--secondary"
              onclick={handleClose}
              disabled={isSubmitting}
            >
              Cancel
            </button>
            <button
              type="submit"
              class="iud-btn iud-btn--primary"
              disabled={isSubmitting}
              aria-busy={isSubmitting}
            >
              {#if isSubmitting}
                <span class="iud-spinner" aria-hidden="true"></span>
                Creating…
              {:else}
                Create Invitation
              {/if}
            </button>
          </footer>
        </form>
      {/if}
    </div>
  </div>
{/if}

<style>
  .iud-overlay {
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

  .iud-modal {
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

  .iud-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 18px 20px;
    border-bottom: 1px solid var(--border-default);
  }

  .iud-title {
    font-size: 15px;
    font-weight: 600;
    color: var(--text-primary);
    margin: 0;
  }

  .iud-close {
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

  .iud-close:hover {
    background: var(--bg-elevated);
    border-color: var(--border-default);
    color: var(--text-primary);
  }

  .iud-form {
    display: flex;
    flex-direction: column;
  }

  .iud-body {
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 10px;
  }

  .iud-label {
    font-size: 12px;
    font-weight: 500;
    color: var(--text-secondary);
    margin-top: 4px;
  }

  .iud-input {
    height: 34px;
    padding: 0 10px;
    border: 1px solid var(--border-default);
    border-radius: var(--radius-sm);
    background: var(--bg-surface);
    color: var(--text-primary);
    font-size: 13px;
    transition: border-color 120ms ease;
  }

  .iud-input:focus {
    outline: none;
    border-color: var(--accent-primary);
  }

  .iud-input:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }

  .iud-input--mono {
    font-family: var(--font-mono);
    font-size: 11px;
  }

  .iud-token-row {
    display: flex;
    gap: 8px;
    align-items: center;
  }

  .iud-token-row .iud-input {
    flex: 1;
  }

  .iud-success-line {
    font-size: 13px;
    color: var(--text-secondary);
    margin: 0 0 4px;
    line-height: 1.5;
  }

  .iud-success-line strong {
    color: var(--text-primary);
    font-weight: 600;
  }

  .iud-hint {
    font-size: 11px;
    color: var(--text-tertiary);
    margin: 6px 0 0;
    line-height: 1.5;
  }

  .iud-hint code {
    font-family: var(--font-mono);
    font-size: 10px;
    padding: 1px 4px;
    background: var(--bg-elevated);
    border-radius: 3px;
    word-break: break-all;
  }

  .iud-error {
    margin-top: 4px;
    padding: 8px 10px;
    border-radius: var(--radius-sm);
    background: rgba(239, 68, 68, 0.1);
    border: 1px solid rgba(239, 68, 68, 0.3);
    color: #fca5a5;
    font-size: 12px;
  }

  .iud-footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 14px 20px;
    border-top: 1px solid var(--border-default);
    background: var(--bg-secondary);
  }

  .iud-btn {
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

  .iud-btn--secondary {
    background: transparent;
    border-color: var(--border-default);
    color: var(--text-secondary);
  }

  .iud-btn--secondary:hover:not(:disabled) {
    background: var(--bg-elevated);
    border-color: var(--border-hover);
    color: var(--text-primary);
  }

  .iud-btn--primary {
    background: rgba(59, 130, 246, 0.2);
    border-color: rgba(59, 130, 246, 0.5);
    color: #93c5fd;
  }

  .iud-btn--primary:hover:not(:disabled) {
    background: rgba(59, 130, 246, 0.3);
    border-color: rgba(59, 130, 246, 0.7);
    color: #bfdbfe;
  }

  .iud-btn:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }

  .iud-spinner {
    width: 12px;
    height: 12px;
    border: 2px solid rgba(147, 197, 253, 0.3);
    border-top-color: #93c5fd;
    border-radius: 50%;
    animation: iud-spin 0.7s linear infinite;
  }

  @keyframes iud-spin {
    to { transform: rotate(360deg); }
  }
</style>
