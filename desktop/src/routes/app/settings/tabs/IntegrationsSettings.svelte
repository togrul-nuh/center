<!-- src/routes/app/settings/tabs/IntegrationsSettings.svelte -->
<script lang="ts">
  import { onMount } from 'svelte';
  import { integrationsStore } from '$lib/stores/integrations.svelte';
  import ConnectIntegrationDialog from '$lib/components/integrations/ConnectIntegrationDialog.svelte';

  // Catalog of integrations we expose in this UI. The `slug` matches the
  // backend identifier used by POST /api/v1/integrations/:slug/connect.
  const INTEGRATIONS = [
    {
      name: 'GitHub',
      slug: 'github',
      desc: 'Push commits, open PRs, manage issues.',
      icon: '⬡',
      tokenLabel: 'Personal Access Token',
      tokenHint: 'Create one at github.com/settings/tokens. Needs repo + read:org scopes.',
    },
    {
      name: 'Linear',
      slug: 'linear',
      desc: 'Sync issues and projects bidirectionally.',
      icon: '◈',
      tokenLabel: 'API Key',
      tokenHint: 'Create one at linear.app/settings/api.',
    },
    {
      name: 'Slack',
      slug: 'slack',
      desc: 'Send notifications and receive commands.',
      icon: '◎',
      tokenLabel: 'Bot Token',
      tokenHint: 'Find it under "OAuth & Permissions" in your Slack app config (starts with xoxb-).',
    },
    {
      name: 'Notion',
      slug: 'notion',
      desc: 'Read and write documents and databases.',
      icon: '⬢',
      tokenLabel: 'Integration Token',
      tokenHint: 'Create one at notion.so/my-integrations (starts with secret_).',
    },
    {
      name: 'Jira',
      slug: 'jira',
      desc: 'Sync issues from Atlassian Jira projects.',
      icon: '◇',
      tokenLabel: 'API Token',
      tokenHint: 'Create one at id.atlassian.com/manage-profile/security/api-tokens.',
    },
    {
      name: 'Datadog',
      slug: 'datadog',
      desc: 'Ingest metrics and alert events.',
      icon: '⬡',
      tokenLabel: 'API Key',
      tokenHint: 'Find it under Organization Settings → API Keys in Datadog.',
    },
  ] as const;

  type IntegrationCatalogEntry = (typeof INTEGRATIONS)[number];

  let dialogOpen = $state(false);
  let activeIntegration = $state<IntegrationCatalogEntry | null>(null);

  onMount(() => {
    void integrationsStore.fetchIntegrations();
  });

  function statusOf(slug: string): 'connected' | 'disconnected' | 'error' {
    const match = integrationsStore.integrations.find((i) => i.slug === slug);
    if (!match) return 'disconnected';
    const raw = match as unknown as Record<string, unknown>;
    if (raw.connected === true) return 'connected';
    if (match.status === 'connected') return 'connected';
    if (match.status === 'error') return 'error';
    return 'disconnected';
  }

  function openConnect(int: IntegrationCatalogEntry) {
    activeIntegration = int;
    dialogOpen = true;
  }

  function closeConnect() {
    dialogOpen = false;
    activeIntegration = null;
  }

  async function disconnect(slug: string) {
    await integrationsStore.disconnect(slug);
  }
</script>

<section class="stg-section">
  <h2 class="stg-section-title">Integrations</h2>
  <p class="stg-section-desc">Connect external services to extend agent capabilities.</p>

  <div class="stg-integration-list">
    {#each INTEGRATIONS as int (int.slug)}
      {@const status = statusOf(int.slug)}
      <div class="stg-int-card">
        <div class="stg-int-icon" aria-hidden="true">{int.icon}</div>
        <div class="stg-int-body">
          <span class="stg-int-name">{int.name}</span>
          <span class="stg-int-desc">{int.desc}</span>
        </div>
        <div class="stg-int-status">
          <span class="stg-int-dot stg-int-dot--{status}"></span>
          <span class="stg-int-label">{status}</span>
        </div>
        {#if status === 'connected'}
          <button
            class="stg-int-btn stg-int-btn--secondary"
            onclick={() => disconnect(int.slug)}
            aria-label="Disconnect {int.name}"
            type="button"
          >
            Disconnect
          </button>
        {:else}
          <button
            class="stg-int-btn"
            onclick={() => openConnect(int)}
            aria-label="Connect {int.name}"
            type="button"
          >
            Connect
          </button>
        {/if}
      </div>
    {/each}
  </div>
</section>

{#if activeIntegration}
  <ConnectIntegrationDialog
    open={dialogOpen}
    integrationName={activeIntegration.name}
    integrationSlug={activeIntegration.slug}
    tokenLabel={activeIntegration.tokenLabel}
    tokenHint={activeIntegration.tokenHint}
    onClose={closeConnect}
  />
{/if}

<style>
  .stg-section { max-width: 640px; }

  .stg-section-title {
    font-size: 15px;
    font-weight: 600;
    color: var(--text-primary);
    margin: 0 0 4px;
  }

  .stg-section-desc {
    font-size: 13px;
    color: var(--text-secondary);
    margin: 0 0 16px;
  }

  .stg-integration-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 16px;
  }

  .stg-int-card {
    display: flex;
    align-items: center;
    gap: 14px;
    padding: 14px 16px;
    background: var(--bg-surface);
    border: 1px solid var(--border-default);
    border-radius: var(--radius-md);
    transition: border-color var(--transition-fast);
  }

  .stg-int-card:hover { border-color: var(--border-hover); }

  .stg-int-icon {
    font-size: 18px;
    width: 32px;
    height: 32px;
    display: flex;
    align-items: center;
    justify-content: center;
    background: var(--bg-elevated);
    border: 1px solid var(--border-default);
    border-radius: var(--radius-sm);
    flex-shrink: 0;
    color: var(--text-secondary);
  }

  .stg-int-body {
    flex: 1;
    min-width: 0;
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .stg-int-name {
    font-size: 13px;
    font-weight: 500;
    color: var(--text-primary);
  }

  .stg-int-desc {
    font-size: 12px;
    color: var(--text-tertiary);
  }

  .stg-int-status {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-shrink: 0;
  }

  .stg-int-dot {
    width: 6px;
    height: 6px;
    border-radius: 50%;
    flex-shrink: 0;
  }

  .stg-int-dot--connected    { background: var(--accent-success); }
  .stg-int-dot--disconnected { background: var(--text-muted); }
  .stg-int-dot--error        { background: var(--accent-error); }

  .stg-int-label {
    font-size: 12px;
    color: var(--text-tertiary);
    text-transform: capitalize;
  }

  .stg-int-btn {
    padding: 5px 12px;
    font-size: 12px;
    font-weight: 500;
    color: var(--accent-primary);
    background: rgba(59, 130, 246, 0.08);
    border: 1px solid rgba(59, 130, 246, 0.2);
    border-radius: var(--radius-sm);
    cursor: pointer;
    flex-shrink: 0;
    transition: background var(--transition-fast);
  }

  .stg-int-btn:hover { background: rgba(59, 130, 246, 0.15); }

  .stg-int-btn--secondary {
    color: var(--text-secondary);
    background: transparent;
    border-color: var(--border-default);
  }

  .stg-int-btn--secondary:hover {
    background: var(--bg-elevated);
    border-color: var(--border-hover);
    color: var(--text-primary);
  }
</style>
