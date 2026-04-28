<script>
  import { tick } from "svelte";
  import { onMount, onDestroy } from "svelte";
  import { createClient } from "@supabase/supabase-js";

  const SUPABASE_URL = import.meta.env.VITE_SUPABASE_URL;
  const SUPABASE_ANON_KEY = import.meta.env.VITE_SUPABASE_ANON_KEY;
  const supabase = createClient(SUPABASE_URL, SUPABASE_ANON_KEY);

  const RANKS = [
    {
      name: "Bronze",
      min: 0,
      icon: "ranks/elo_bronze.png",
    },
    {
      name: "Prata",
      min: 1200,
      icon: "ranks/elo_prata.png",
    },
    {
      name: "Ouro",
      min: 1400,
      icon: "ranks/elo_ouro.png",
    },
    {
      name: "Platina",
      min: 1600,
      icon: "ranks/elo_platina.png",
    },
    {
      name: "Diamante",
      min: 1800,
      icon: "ranks/elo_diamante.png",
    },
    {
      name: "Mestre",
      min: 2000,
      icon: "ranks/elo_mestre.png",
    },
  ];

  function getRank(elo) {
    let currentRank = RANKS[0];

    for (const rank of RANKS) {
      if (elo >= rank.min) {
        currentRank = rank;
      }
    }

    return currentRank;
  }

  function expectedScore(Ra, Rb) {
    return 1 / (1 + Math.pow(10, (Rb - Ra) / 400));
  }

  function updateElo(playerA, playerB, scoreA, K = 64) {
    const Ea = expectedScore(playerA.elo, playerB.elo);
    const Eb = expectedScore(playerB.elo, playerA.elo);

    playerA.elo = Math.round(playerA.elo + K * (scoreA - Ea));
    playerB.elo = Math.round(playerB.elo + K * (1 - scoreA - Eb));
  }

  async function fetchPlayers() {
    const { data, error } = await supabase
      .from("players")
      .select("*")
      .order("elo", { ascending: false });

    if (error) {
      console.error(error);
      return;
    }

    players = data.map((p) => ({
      id: p.id,
      name: p.name,
      win: p.win,
      lose: p.lose,
      total: p.total,
      winRate: p.win_rate,
      elo: p.elo ?? 1000,
    }));
  }

  async function insertPlayer(name) {
    const { data, error } = await supabase
      .from("players")
      .insert({
        name,
        win: 0,
        lose: 0,
        total: 0,
        win_rate: 0,
        elo: 1000,
      })
      .select()
      .single();

    if (error) {
      console.error(error);
      return null;
    }

    return {
      id: data.id,
      name: data.name,
      win: 0,
      lose: 0,
      total: 0,
      winRate: 0,
      elo: data.elo,
    };
  }

  async function updatePlayerStats(player) {
    const { error } = await supabase
      .from("players")
      .update({
        win: player.win,
        lose: player.lose,
        total: player.total,
        win_rate: player.winRate,
        elo: player.elo,
      })
      .eq("id", player.id);

    if (error) console.error(error);
  }

  function handleKeydown(e) {
    if (e.key === "Escape") {
      showAddPlayer = false;
      showAddToQueue = false;
      showRemoveFromQueue = false;
      showMatchResult = false;
    }
  }

  onMount(async () => {
    window.addEventListener("keydown", handleKeydown);
    await fetchPlayers();
  });

  onDestroy(() => {
    window.removeEventListener("keydown", handleKeydown);
  });

  let inputRef = $state.raw(null);
  let prevShowAddPlayer = false;

  $effect(() => {
    if (showAddPlayer && !prevShowAddPlayer) focusInput();
    prevShowAddPlayer = showAddPlayer;
  });

  async function focusInput() {
    await tick();
    inputRef?.focus();
  }

  let players = $state([]);
  let queue = $state([]);

  let showAddPlayer = $state(false);
  let showAddToQueue = $state(false);
  let showRemoveFromQueue = $state(false);
  let showMatchResult = $state(false);

  let newPlayerName = $state("");
  let selectedToAdd = $state([]);
  let selectedToRemove = $state([]);

  let isAddingPlayer = $state(false);
  let isRecordingMatch = $state(false);

  function sortPlayers() {
    players = [...players].sort((a, b) => b.elo - a.elo);
  }

  async function confirmAddPlayer() {
    if (isAddingPlayer) return;

    if (!newPlayerName.trim()) return;

    isAddingPlayer = true;

    const playerName = newPlayerName.trim();

    newPlayerName = "";
    showAddPlayer = false;

    try {
      const newPlayer = await insertPlayer(playerName);

      if (newPlayer) {
        players = [...players, newPlayer];
        sortPlayers();
      }
    } finally {
      isAddingPlayer = false;
    }
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

  function moveUp(i) {
    if (i === 0) return;

    const newQueue = [...queue];

    [newQueue[i - 1], newQueue[i]] = [newQueue[i], newQueue[i - 1]];

    queue = newQueue;
  }

  function moveDown(i) {
    if (i === queue.length - 1) return;

    const newQueue = [...queue];

    [newQueue[i], newQueue[i + 1]] = [newQueue[i + 1], newQueue[i]];

    queue = newQueue;
  }

  async function recordWinner(winnerIndex) {
    if (isRecordingMatch) return;

    isRecordingMatch = true;
    showMatchResult = false;

    try {
      const winner = queue[winnerIndex];
      const loserIndex = winnerIndex === 0 ? 1 : 0;
      const loser = queue[loserIndex];

      winner.win++;
      loser.lose++;

      winner.total = winner.win + winner.lose;
      loser.total = loser.win + loser.lose;

      winner.winRate = winner.win / winner.total;
      loser.winRate = loser.win / loser.total;

      updateElo(winner, loser, 1);

      await Promise.all([updatePlayerStats(winner), updatePlayerStats(loser)]);

      const newQueue = [...queue];

      if (winnerIndex === 0) {
        newQueue.splice(1, 1);
        newQueue.push(loser);
      } else {
        newQueue.splice(0, 1);
        newQueue.push(loser);
      }

      queue = newQueue;

      sortPlayers();
    } finally {
      isRecordingMatch = false;
    }
  }

  function toggleSelect(arr, name) {
    return arr.includes(name) ? arr.filter((n) => n !== name) : [...arr, name];
  }
</script>

<div class="app">
  <main>
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
            <div class="reorder-btns">
              <button
                class="arrow-btn"
                onclick={() => moveUp(i)}
                disabled={i === 0}>▲</button
              >
              <button
                class="arrow-btn"
                onclick={() => moveDown(i)}
                disabled={i === queue.length - 1}>▼</button
              >
            </div>
          </div>
          {#if i === 0 && queue.length > 1}
            <div class="vs-divider">vs</div>
          {/if}
        {/each}
      </div>

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
                <th>Elo</th>
              </tr>
            </thead>
            <tbody>
              {#each players as p, i}
                {@const rank = getRank(p.elo)}

                <tr>
                  <td>{i + 1}</td>

                  <td>
                    <div class="player-name">
                      <img src={rank.icon} alt={rank.name} class="rank-icon" />

                      <div class="player-info">
                        <span>{p.name}</span>
                        <span class="rank-label">{rank.name}</span>
                      </div>
                    </div>
                  </td>

                  <td>{p.win}</td>
                  <td>{p.lose}</td>
                  <td>{p.total}</td>
                  <td>{(p.winRate * 100).toFixed(0)}%</td>
                  <td>{p.elo}</td>
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
        <button class="btn btn-outline" onclick={() => (showAddPlayer = false)}
          >Cancelar</button
        >
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
        <button class="btn btn-outline" onclick={() => (showAddToQueue = false)}
          >Cancelar</button
        >
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
          onclick={() => (showRemoveFromQueue = false)}>Cancelar</button
        >
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
        <button class="winner-btn" onclick={() => recordWinner(0)}
          >{queue[0].name}</button
        >
        <span class="vs-text">vs</span>
        <button class="winner-btn" onclick={() => recordWinner(1)}
          >{queue[1].name}</button
        >
      </div>
      <button class="btn btn-outline" onclick={() => (showMatchResult = false)}
        >Cancelar</button
      >
    </div>
  </div>
{/if}

<style>
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

  .app {
    max-width: 1200px;
    margin: 0 auto;
    padding: 24px 16px 48px;
  }

  main {
    display: grid;
    grid-template-columns: 1.5fr 600px;
    gap: 16px;
    align-items: stretch;
  }

  @media (max-width: 560px) {
    main {
      grid-template-columns: 1fr;
    }
  }

  .panel {
    background: #fff;
    border: 1px solid #e4e4e4;
    border-radius: 14px;
    padding: 20px;

    display: flex;
    flex-direction: column;
    gap: 14px;

    height: 80vh;
    min-height: 0;
  }

  h2 {
    font-size: 0.8rem;
    font-weight: 600;
    letter-spacing: 0.08em;
    text-transform: uppercase;
    color: #888;

    flex-shrink: 0;
  }

  .empty {
    flex: 1;

    display: flex;
    align-items: center;
    justify-content: center;

    font-size: 0.85rem;
    color: #aaa;

    text-align: center;
  }

  .queue-list,
  .players-table {
    flex: 1;
    min-height: 0;

    overflow-y: auto;
    overflow-x: auto;

    padding-right: 4px;
  }

  .queue-list::-webkit-scrollbar,
  .players-table::-webkit-scrollbar {
    width: 8px;
    height: 8px;
  }

  .queue-list::-webkit-scrollbar-thumb,
  .players-table::-webkit-scrollbar-thumb {
    background: #d0d0d0;
    border-radius: 999px;
  }

  .queue-list::-webkit-scrollbar-thumb:hover,
  .players-table::-webkit-scrollbar-thumb:hover {
    background: #b8b8b8;
  }

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

    flex-shrink: 0;
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

  .vs-divider {
    text-align: center;
    font-size: 0.75rem;
    font-weight: 700;
    color: #5cb97d;
    letter-spacing: 0.1em;
    margin: -2px 0 4px;

    flex-shrink: 0;
  }

  .reorder-btns {
    display: flex;
    flex-direction: column;
    gap: 2px;
  }

  .arrow-btn {
    display: flex;
    align-items: center;
    justify-content: center;

    width: 22px;
    height: 18px;

    border: 1px solid #ddd;
    border-radius: 4px;

    background: transparent;

    color: #888;
    font-size: 0.6rem;

    cursor: pointer;

    padding: 0;

    line-height: 1;

    transition:
      background 0.1s,
      color 0.1s;
  }

  .arrow-btn:hover:not(:disabled) {
    background: #f0f0f0;
    color: #111;
  }

  .arrow-btn:disabled {
    opacity: 0.2;
    cursor: not-allowed;
  }

  .btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    gap: 6px;

    width: 100%;

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

    flex-shrink: 0;
  }

  .btn-match:hover {
    background: #4aa86c;
  }

  .queue-actions,
  .player-actions {
    display: flex;
    flex-direction: column;
    gap: 8px;

    margin-top: auto;

    flex-shrink: 0;
  }

  .overlay {
    position: fixed;
    inset: 0;

    background: rgba(0, 0, 0, 0.25);

    z-index: 100;

    border: none;
    padding: 0;
    margin: 0;

    cursor: pointer;
  }

  .modal-root {
    position: fixed;
    inset: 0;
    z-index: 100;
  }

  .modal {
    position: fixed;

    top: 50%;
    left: 50%;

    transform: translate(-50%, -50%);

    width: 100%;
    max-width: 360px;

    background: #fff;

    border-radius: 16px;

    padding: 28px 24px 22px;

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

  .players-table thead th {
    position: sticky;
    top: 0;

    background: #fff;

    z-index: 1;
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

  .player-name {
    display: flex;
    align-items: center;
    gap: 10px;
  }

  .player-info {
    display: flex;
    flex-direction: column;
    gap: 0;
  }

  .rank-icon {
    width: 28px;
    height: 28px;

    object-fit: contain;

    flex-shrink: 0;
  }

  .rank-label {
    font-size: 0.68rem;
    color: #888;
    font-weight: 600;

    text-transform: uppercase;

    letter-spacing: 0.06em;
  }
</style>
