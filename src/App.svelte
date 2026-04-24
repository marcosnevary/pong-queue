<script>
  import { tick } from "svelte";
  import { onMount, onDestroy } from "svelte";

  function handleKeydown(e) {
    if (e.key === "Escape") {
      showAddPlayer = false;
      showAddToQueue = false;
      showRemoveFromQueue = false;
      showMatchResult = false;
    }
  }

  onMount(() => {
    window.addEventListener("keydown", handleKeydown);
  });

  onDestroy(() => {
    window.removeEventListener("keydown", handleKeydown);
  });

  let inputRef = $state.raw(null);
  let prevShowAddPlayer = false;

  $effect(() => {
    if (showAddPlayer && !prevShowAddPlayer) {
      focusInput();
    }

    prevShowAddPlayer = showAddPlayer;
  });

  async function focusInput() {
    await tick();
    inputRef?.focus();
  }

  // ─── State ────────────────────────────────────────────────────────────────
  let players = $state([]);
  let queue = $state([]);

  // Modal state
  let showAddPlayer = $state(false);
  let showAddToQueue = $state(false);
  let showRemoveFromQueue = $state(false);
  let showMatchResult = $state(false);

  let newPlayerName = $state("");
  let selectedToAdd = $state([]);
  let selectedToRemove = $state([]);

  // ─── Helpers ──────────────────────────────────────────────────────────────
  function sortPlayers() {
    players = [...players].sort((a, b) => b.winRate - a.winRate);
  }

  function addPlayer(name) {
    if (!name.trim()) return;
    if (players.find((p) => p.name === name.trim())) return;
    players = [
      ...players,
      { name: name.trim(), win: 0, lose: 0, total: 0, winRate: 0 },
    ];
    sortPlayers();
  }

  function confirmAddPlayer() {
    addPlayer(newPlayerName);
    newPlayerName = "";
    showAddPlayer = false;
  }

  function confirmAddToQueue() {
    const toAdd = players.filter(
      (p) => selectedToAdd.includes(p.name) && !queue.includes(p),
    );
    queue = [...queue, ...toAdd];
    selectedToAdd = [];
    showAddToQueue = false;
  }

  function confirmRemoveFromQueue() {
    queue = queue.filter((p) => !selectedToRemove.includes(p.name));
    selectedToRemove = [];
    showRemoveFromQueue = false;
  }

  function clearQueue() {
    queue = [];
  }

  function recordWinner(winnerIndex) {
    const winner = queue[winnerIndex];
    const loserIndex = winnerIndex === 0 ? 1 : 0;
    const loser = queue[loserIndex];

    // Update stats
    winner.win++;
    winner.total = winner.win + winner.lose;
    winner.winRate = winner.win / winner.total;

    loser.lose++;
    loser.total = loser.win + loser.lose;
    loser.winRate = loser.win / loser.total;

    // Winner stays at front, loser goes to back
    const newQueue = [...queue];
    if (winnerIndex === 0) {
      // player 0 won → remove player 1, append to end
      newQueue.splice(1, 1);
      newQueue.push(loser);
    } else {
      // player 1 won → remove player 0, append to end, player 1 moves to front
      newQueue.splice(0, 1);
      newQueue.push(loser);
    }
    queue = newQueue;

    sortPlayers();
    showMatchResult = false;
  }

  function toggleSelect(arr, name) {
    return arr.includes(name) ? arr.filter((n) => n !== name) : [...arr, name];
  }
</script>

<!-- ─── Main Layout ──────────────────────────────────────────────────────── -->
<div class="app">
  <main>
    <!-- Queue Column -->
    <section class="panel queue-panel">
      <h2>Fila</h2>

      <div class="queue-list">
        {#if queue.length === 0}
          <p class="empty">Nenhum jogador na fila.</p>
        {/if}

        {#each queue as player, i}
          <div class="queue-card" class:active={i < 2}>
            <span class="pos">{i + 1}</span>
            <span class="pname">{player.name}</span>
            {#if i < 2}
              <span class="badge">{i === 0 ? "▶" : "◀"}</span>
            {/if}
          </div>
          {#if i === 0 && queue.length > 1}
            <div class="vs-divider">vs</div>
          {/if}
        {/each}
      </div>

      <!-- Match button -->
      {#if queue.length >= 2}
        <button class="btn btn-match" onclick={() => (showMatchResult = true)}>
          Registrar resultado
        </button>
      {/if}

      <div class="queue-actions">
        <button
          class="btn"
          onclick={() => {
            selectedToAdd = [];
            showAddToQueue = true;
          }}
        >
          Adicionar à fila
        </button>
        <button
          class="btn btn-outline"
          onclick={() => {
            selectedToRemove = [];
            showRemoveFromQueue = true;
          }}
        >
          Remover da fila
        </button>
        <button class="btn btn-danger" onclick={clearQueue}>
          Limpar fila
        </button>
      </div>
    </section>

    <!-- Players Column -->
    <section class="panel players-panel">
      <h2>Jogadores</h2>

      <div class="players-table">
        {#if players.length === 0}
          <p class="empty">Nenhum jogador cadastrado.</p>
        {:else}
          <table>
            <thead>
              <tr>
                <th>#</th>
                <th>Nome</th>
                <th>Vitórias</th>
                <th>Derrotas</th>
                <th>Total</th>
                <th>Win Rate</th>
              </tr>
            </thead>
            <tbody>
              {#each players as p, i}
                <tr>
                  <td>{i + 1}</td>
                  <td>{p.name}</td>
                  <td>{p.win}</td>
                  <td>{p.lose}</td>
                  <td>{p.total}</td>
                  <td>{(p.winRate * 100).toFixed(0)}%</td>
                </tr>
              {/each}
            </tbody>
          </table>
        {/if}
      </div>

      <div class="player-actions">
        <button
          class="btn"
          onclick={() => {
            newPlayerName = "";
            showAddPlayer = true;
          }}
        >
          Adicionar jogador
        </button>
      </div>
    </section>
  </main>
</div>

{#if showAddPlayer}
  <div class="modal-root">
    <button
      type="button"
      class="overlay"
      onclick={() => (showAddPlayer = false)}
      aria-label="Fechar modal"
    ></button>

    <div class="modal">
      <h3>Novo jogador</h3>
      <input
        bind:this={inputRef}
        type="text"
        placeholder="Nome do jogador"
        bind:value={newPlayerName}
        onkeydown={(e) => e.key === "Enter" && confirmAddPlayer()}
      />
      <div class="modal-actions">
        <button class="btn" onclick={confirmAddPlayer}>Adicionar</button>
        <button class="btn btn-outline" onclick={() => (showAddPlayer = false)}>
          Cancelar
        </button>
      </div>
    </div>
  </div>
{/if}

{#if showAddToQueue}
  <div class="modal-root">
    <button
      type="button"
      class="overlay"
      onclick={() => (showAddToQueue = false)}
      aria-label="Fechar modal"
    ></button>

    <div class="modal">
      <h3>Adicionar à fila</h3>

      {#if players.filter((p) => !queue.includes(p)).length === 0}
        <p class="empty">Todos os jogadores já estão na fila.</p>
      {:else}
        <div class="select-list">
          {#each players.filter((p) => !queue.includes(p)) as p}
            <label class="select-item">
              <input
                type="checkbox"
                checked={selectedToAdd.includes(p.name)}
                onchange={() =>
                  (selectedToAdd = toggleSelect(selectedToAdd, p.name))}
              />
              {p.name}
            </label>
          {/each}
        </div>
      {/if}

      <div class="modal-actions">
        <button
          class="btn"
          onclick={confirmAddToQueue}
          disabled={selectedToAdd.length === 0}
        >
          Adicionar ({selectedToAdd.length})
        </button>

        <button
          class="btn btn-outline"
          onclick={() => (showAddToQueue = false)}
        >
          Cancelar
        </button>
      </div>
    </div>
  </div>
{/if}

{#if showRemoveFromQueue}
  <div class="modal-root">
    <button
      type="button"
      class="overlay"
      onclick={() => (showRemoveFromQueue = false)}
      aria-label="Fechar modal"
    ></button>

    <div class="modal">
      <h3>Remover da fila</h3>

      {#if queue.length === 0}
        <p class="empty">Fila vazia.</p>
      {:else}
        <div class="select-list">
          {#each queue as p}
            <label class="select-item">
              <input
                type="checkbox"
                checked={selectedToRemove.includes(p.name)}
                onchange={() =>
                  (selectedToRemove = toggleSelect(selectedToRemove, p.name))}
              />
              {p.name}
            </label>
          {/each}
        </div>
      {/if}

      <div class="modal-actions">
        <button
          class="btn btn-danger"
          onclick={confirmRemoveFromQueue}
          disabled={selectedToRemove.length === 0}
        >
          Remover ({selectedToRemove.length})
        </button>

        <button
          class="btn btn-outline"
          onclick={() => (showRemoveFromQueue = false)}
        >
          Cancelar
        </button>
      </div>
    </div>
  </div>
{/if}

{#if showMatchResult && queue.length >= 2}
  <div class="modal-root">
    <button
      type="button"
      class="overlay"
      onclick={() => (showMatchResult = false)}
      aria-label="Fechar modal"
    ></button>

    <div class="modal match-modal">
      <h3>Quem ganhou?</h3>

      <div class="match-choice">
        <button class="winner-btn" onclick={() => recordWinner(0)}>
          {queue[0].name}
        </button>

        <span class="vs-text">vs</span>

        <button class="winner-btn" onclick={() => recordWinner(1)}>
          {queue[1].name}
        </button>
      </div>

      <button class="btn btn-outline" onclick={() => (showMatchResult = false)}>
        Cancelar
      </button>
    </div>
  </div>
{/if}

<style>
  /* ─── Reset & Base ─────────────────────────────────────────────────────── */
  :global(*, *::before, *::after) {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
  }
  :global(body) {
    font-family: "Geist", "DM Sans", ui-sans-serif, system-ui, sans-serif;
    background: #f8f8f8;
    color: #111;
    min-height: 100vh;
  }

  /* ─── App Shell ────────────────────────────────────────────────────────── */
  .app {
    max-width: 1200px;
    margin: 0 auto;
    padding: 24px 16px 48px;
  }

  /* ─── Two-column Layout ────────────────────────────────────────────────── */
  main {
    display: grid;
    grid-template-columns: 1.5fr 520px;
    gap: 16px;
    align-items: start;
  }

  @media (max-width: 560px) {
    main {
      grid-template-columns: 1fr;
    }
  }

  /* ─── Panel ────────────────────────────────────────────────────────────── */
  .panel {
    background: #fff;
    border: 1px solid #e4e4e4;
    border-radius: 14px;
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 14px;
  }

  h2 {
    font-size: 0.8rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: #888;
  }

  .empty {
    font-size: 0.85rem;
    color: #aaa;
    text-align: center;
    padding: 12px 0;
  }

  /* ─── Queue List ───────────────────────────────────────────────────────── */
  .queue-list {
    display: flex;
    flex-direction: column;
    gap: 0;
  }

  .queue-card {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 10px 12px;
    border-radius: 10px;
    border: 1.5px solid #e4e4e4;
    background: #fafafa;
    margin-bottom: 6px;
    transition: background 0.15s;
  }
  .queue-card.active {
    background: #e6f5ec;
    border-color: #5cb97d;
  }
  .queue-card .pos {
    font-size: 0.72rem;
    color: #aaa;
    font-weight: 600;
    min-width: 18px;
  }
  .queue-card .pname {
    font-size: 0.95rem;
    font-weight: 600;
    flex: 1;
  }
  .queue-card .badge {
    font-size: 0.75rem;
    color: #5cb97d;
  }

  .vs-divider {
    text-align: center;
    font-size: 0.75rem;
    font-weight: 700;
    color: #5cb97d;
    letter-spacing: 0.1em;
    margin: -2px 0 4px;
  }

  /* ─── Buttons ──────────────────────────────────────────────────────────── */
  .btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
    padding: 9px 16px;
    border-radius: 8px;
    border: 1.5px solid #111;
    background: #111;
    color: #fff;
    font-size: 0.83rem;
    font-weight: 600;
    cursor: pointer;
    transition:
      background 0.12s,
      transform 0.08s;
    width: 100%;
  }
  .btn:hover {
    background: #333;
  }
  .btn:active {
    transform: scale(0.98);
  }
  .btn:disabled {
    opacity: 0.4;
    cursor: not-allowed;
  }

  .btn-outline {
    background: transparent;
    color: #111;
  }
  .btn-outline:hover {
    background: #f0f0f0;
  }

  .btn-danger {
    background: transparent;
    color: #c0392b;
    border-color: #c0392b;
  }
  .btn-danger:hover {
    background: #fdf0ef;
  }

  .btn-match {
    background: #5cb97d;
    border-color: #5cb97d;
    color: #fff;
  }
  .btn-match:hover {
    background: #4aa86c;
  }

  .queue-actions,
  .player-actions {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 4px;
  }

  /* ─── Modal ────────────────────────────────────────────────────────────── */
  /* ─── Modal ────────────────────────────────────────────────────────────── */
  .overlay {
    position: fixed;
    inset: 0;
    background: rgba(0, 0, 0, 0.25);
    z-index: 100;

    /* botão reset */
    border: none;
    padding: 0;
    margin: 0;
    cursor: pointer;
  }

  /* NOVO: container raiz do modal */
  .modal-root {
    position: fixed;
    inset: 0;
    z-index: 100;
  }

  /* modal agora centralizado manualmente */
  .modal {
    position: fixed;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);

    background: #fff;
    border-radius: 16px;
    padding: 28px 24px 22px;
    width: 100%;
    max-width: 360px;
    box-shadow: 0 8px 40px rgba(0, 0, 0, 0.12);

    display: flex;
    flex-direction: column;
    gap: 16px;

    z-index: 101;
  }

  .modal h3 {
    font-size: 1.05rem;
    font-weight: 700;
    letter-spacing: -0.01em;
  }

  .modal input[type="text"] {
    width: 100%;
    padding: 10px 12px;
    border: 1.5px solid #e0e0e0;
    border-radius: 8px;
    font-size: 0.95rem;
    outline: none;
    transition: border-color 0.15s;
  }
  .modal input[type="text"]:focus {
    border-color: #111;
  }

  .modal-actions {
    display: flex;
    gap: 8px;
  }
  .modal-actions .btn {
    flex: 1;
  }

  /* Select list */
  .select-list {
    display: flex;
    flex-direction: column;
    gap: 6px;
    max-height: 240px;
    overflow-y: auto;
  }
  .select-item {
    display: flex;
    align-items: center;
    gap: 10px;
    padding: 9px 12px;
    border-radius: 8px;
    border: 1.5px solid #e8e8e8;
    cursor: pointer;
    font-size: 0.9rem;
    font-weight: 500;
    transition: background 0.1s;
  }
  .select-item:hover {
    background: #f5f5f5;
  }
  .select-item input {
    accent-color: #111;
    width: 15px;
    height: 15px;
  }

  /* Match modal */
  .match-modal {
    text-align: center;
  }
  .match-choice {
    display: flex;
    align-items: center;
    gap: 12px;
  }
  .winner-btn {
    flex: 1;
    padding: 16px 10px;
    border-radius: 12px;
    border: 2px solid #5cb97d;
    background: #e6f5ec;
    color: #2d7a4f;
    font-size: 0.9rem;
    font-weight: 700;
    cursor: pointer;
    transition:
      background 0.12s,
      transform 0.08s;
  }
  .winner-btn:hover {
    background: #d0eddb;
    transform: scale(1.02);
  }
  .vs-text {
    font-size: 0.85rem;
    font-weight: 700;
    color: #aaa;
  }

  .players-table table {
    width: 100%;
    border-collapse: collapse;
    font-size: 0.85rem;
  }

  .players-table th {
    text-align: left;
    font-size: 0.7rem;
    text-transform: uppercase;
    color: #888;
    padding: 6px 4px;
    letter-spacing: 0.1em;
  }

  .players-table td {
    padding: 8px 4px;
    border-top: 1px solid #eee;
  }

  .players-table tbody tr:hover {
    background: #f8f8f8;
  }

  .players-table table th,
  .players-table table td {
    text-align: left;
  }
</style>
