<script lang="ts">
  import type { Card } from '#lib/types';
  import { app, activeNode, briefSummary, inheritedCards, openNode, ownCards, unconfirmed } from '#lib/store.svelte';
  import { focusTrap } from '#lib/focus';
  import Carta from './Carta.svelte';
  import {
    ArrowUpRight,
    FileText,
    GitFork,
    Lightbulb,
    ScrollText,
    Sparkles,
    UserRound,
    X
  } from '@lucide/svelte';

  let archiveScope = $state<'heredadas' | 'propias' | 'todas'>('heredadas');
  let archiveQuery = $state('');

  const node = $derived(activeNode());
  const own = $derived(ownCards(node));
  const inherited = $derived(inheritedCards(node));
  const activeScopedCount = $derived(
    inherited.filter((c) => c.scope === 'global' || c.scope === 'especie').length
  );
  const archivePool = $derived<Card[]>(
    archiveScope === 'propias' ? own : archiveScope === 'heredadas' ? inherited : [...own, ...inherited]
  );
  const archiveFiltered = $derived.by(() => {
    const q = archiveQuery.trim().toLowerCase();
    return archivePool.filter((c) => {
      if (q && !`${c.text} ${c.type} ${c.scope}`.toLowerCase().includes(q)) return false;
      return true;
    });
  });
  const archiveShown = $derived(archiveFiltered.slice(0, app.ledgerLimit));
  const archiveScopes: { id: 'heredadas' | 'propias' | 'todas'; label: string }[] = [
    { id: 'heredadas', label: 'De otras respuestas' },
    { id: 'propias', label: 'De esta respuesta' },
    { id: 'todas', label: 'Todo' }
  ];
  const assumptions = $derived(
    node?.cards.filter((c) => c.type === 'supuesto' && c.source !== 'usuario' && c.state !== 'ignorado') ?? []
  );
  const proposals = $derived(
    node?.cards.filter((c) => c.source === 'ia' && c.type !== 'supuesto' && c.state !== 'ignorado') ?? []
  );
  const userCards = $derived(
    node?.cards.filter((c) => c.source === 'usuario' && c.state !== 'ignorado') ?? []
  );
  const brief = $derived(briefSummary(node?.brief));
  const typeLabel: Record<string, string> = {
    hecho: 'Dato',
    supuesto: 'Suposición',
    preferencia: 'Preferencia',
    artefacto: 'Material'
  };

  const branches = $derived(app.session.branches);
  const tabs = [
    { id: 'lamina' as const, label: 'Esta respuesta', icon: FileText },
    { id: 'heredado' as const, label: 'Lo que recuerda', icon: GitFork }
  ];

  function setTab(tab: 'lamina' | 'heredado') {
    app.ledgerTab = tab;
    app.ledgerLimit = 30;
  }

  function close() {
    app.ledgerOpen = false;
  }

  function go(id: string) {
    openNode(id);
    close();
  }
</script>

{#if app.ledgerOpen}
  <div class="overlay">
    <button class="scrim" aria-label="Cerrar Memoria" onclick={close}></button>
    <div
      class="modal ledger"
      role="dialog"
      aria-modal="true"
      aria-label="Memoria"
      tabindex="-1"
      use:focusTrap
    >
      <header class="ledger-head">
        <span class="modal-icon"><ScrollText size={17} /></span>
        <div>
          <h2>Memoria</h2>
          <p>{app.session.title} · qué recuerda la IA y de dónde salió esta respuesta</p>
        </div>
        <button class="icon-btn" aria-label="Cerrar Memoria" onclick={close}><X size={18} /></button>
      </header>

      <div class="facets">
        {#each tabs as t (t.id)}
          {@const TabIcon = t.icon}
          <button
            class="facet"
            class:on={app.ledgerTab === t.id}
            onclick={() => setTab(t.id)}
          >
            <TabIcon size={12} />
            {t.label}
          </button>
        {/each}
      </div>

      {#if app.ledgerTab === 'lamina'}
        {#if !node}
          <div class="results-empty">Abre una respuesta para ver sus tres voces.</div>
        {:else}
          <div class="ledger-body">
            <section class="voice user">
              <h3><UserRound size={15} /> Tú dijiste</h3>
              <p class="quote">«{node.prompt}»</p>
              <p class="voice-meta">{node.date} · hilo {branches.find((b) => b.id === node.branchId)?.name ?? '—'}</p>
              {#if brief.length}
                <div class="facts">
                  {#each brief as f (f)}<span class="fact">{f}</span>{/each}
                </div>
              {/if}
              {#if userCards.length}
                <div class="ledger-cards">
                  {#each userCards as c (c.id)}<Carta card={c} compact />{/each}
                </div>
              {/if}
            </section>

            <section class="voice ai">
              <h3><Sparkles size={15} /> La IA propuso</h3>
              <p>{node.esencia}</p>
              <p class="voice-meta">
                {node.model} · {node.generatedAt} ·
                {node.actions.length} acciones · {node.params.length} parámetros ·
                {node.sections.length} secciones
              </p>
              {#if proposals.length}
                <ul class="voice-list">
                  {#each proposals as c (c.id)}
                    <li>
                      <span class="tag" class:warn={c.type === 'hecho'}>{typeLabel[c.type] ?? c.type}</span>
                      {c.text}
                    </li>
                  {/each}
                </ul>
              {/if}
              <div class="voice-actions">
                <button class="btn tiny" onclick={() => go(node.id)}>
                  Abrir respuesta
                  <ArrowUpRight size={13} />
                </button>
              </div>
            </section>

            <section class="voice assumed">
              <h3><Lightbulb size={15} /> La IA supuso</h3>
              {#if assumptions.length === 0}
                <p class="voice-meta">No hay supuestos registrados.</p>
              {:else}
                <p class="voice-meta">
                  {unconfirmed(node).length} sin confirmar · {assumptions.length} en total
                </p>
                <div class="ledger-cards">
                  {#each assumptions as c (c.id)}
                    <Carta card={c} />
                  {/each}
                </div>
              {/if}
            </section>
          </div>
        {/if}
      {:else}
        {#if !node}
          <div class="ledger-body">
            <div class="results-empty">Abre una respuesta para revisar qué recuerda la IA.</div>
          </div>
        {:else}
          <div class="ledger-filters">
            <input
              class="modal-input"
              placeholder="Filtrar datos, suposiciones o materiales…"
              aria-label="Filtrar el archivo de contexto"
              bind:value={archiveQuery}
              oninput={() => (app.ledgerLimit = 30)}
            />
          </div>

          <div class="facets">
            {#each archiveScopes as s (s.id)}
              <button
                class="facet"
                class:on={archiveScope === s.id}
                onclick={() => {
                  archiveScope = s.id;
                  app.ledgerLimit = 30;
                }}
              >
                {s.label}
                <span class="facet-count">
                  {s.id === 'heredadas' ? inherited.length : s.id === 'propias' ? own.length : own.length + inherited.length}
                </span>
              </button>
            {/each}
          </div>

          <div class="ledger-body">
            {#if archiveFiltered.length === 0}
              <div class="results-empty">
                <p>No hay recuerdos que coincidan para {node.species}.</p>
                <button
                  class="btn tiny"
                  onclick={() => {
                    archiveScope = 'todas';
                    archiveQuery = '';
                  }}
                >Limpiar filtros</button>
              </div>
            {:else}
              <p class="voice-meta">
                Cartas de esta respuesta y heredadas de otras respuestas, filtrables por alcance y tipo.{activeScopedCount ? ` ${activeScopedCount} reglas heredadas activas.` : ''}
              </p>
              <div class="ledger-cards">
                {#each archiveShown as c (c.id)}
                  <Carta card={c} />
                {/each}
              </div>
              {#if archiveFiltered.length > archiveShown.length}
                <button class="more-btn" onclick={() => (app.ledgerLimit += 30)}>
                  Mostrar 30 más · quedan {archiveFiltered.length - archiveShown.length}
                </button>
              {/if}
            {/if}
          </div>
        {/if}
      {/if}

      <footer class="ledger-foot">
        <span>Confirmar o corregir versiona y marca dependientes como desactualizadas.</span>
      </footer>
    </div>
  </div>
{/if}
