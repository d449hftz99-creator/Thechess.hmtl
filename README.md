# Thechess.hmtl
<!DOCTYPE html>
<html lang="de">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Chess App – Partie starten funktioniert</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.10.3/chess.min.js"></script>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/chessboard.js/1.0.0/chessboard-1.0.0.min.js"></script>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/chessboard.js/1.0.0/chessboard-1.0.0.min.css">
  <style>
    :root {
      --bg: #1a1a1a;
      --card: #2a2a2a;
      --accent: #81b64c;
      --text: #e8e8e8;
      --muted: #aaa;
    }
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body {
      font-family: system-ui, -apple-system, sans-serif;
      background: var(--bg);
      color: var(--text);
      min-height: 100vh;
      padding: 16px;
    }
    .container { max-width: 1200px; margin: 0 auto; }
    header {
      display: flex; justify-content: space-between; align-items: center;
      margin-bottom: 20px; flex-wrap: wrap; gap: 12px;
    }
    h1 { font-size: 1.6rem; color: var(--accent); }
    .nav { display: flex; gap: 8px; flex-wrap: wrap; }
    button, select {
      background: var(--card); color: var(--text); border: 1px solid #444;
      padding: 8px 14px; border-radius: 6px; cursor: pointer; font-size: 0.95rem;
    }
    button:hover, select:hover { border-color: var(--accent); }
    button.primary { background: var(--accent); color: #111; font-weight: 600; }
    button.danger { background: #c44; color: white; }
    button:disabled { opacity: 0.45; cursor: not-allowed; }
    .main { display: grid; grid-template-columns: 1fr 340px; gap: 24px; }
    @media (max-width: 900px) { .main { grid-template-columns: 1fr; } }
    #board { width: 100%; max-width: 560px; margin: 0 auto; }
    .panel {
      background: var(--card); border-radius: 10px; padding: 16px;
      display: flex; flex-direction: column; gap: 12px;
    }
    .panel h3 { font-size: 1.1rem; margin-bottom: 4px; color: var(--accent); }
    .status { font-size: 1rem; color: var(--muted); min-height: 28px; font-weight: 500; }
    .clocks { display: flex; justify-content: space-between; font-size: 1.4rem; font-weight: 600; }
    .clock { background: #333; padding: 6px 12px; border-radius: 6px; min-width: 80px; text-align: center; }
    .clock.active { background: var(--accent); color: #111; }
    .moves { max-height: 160px; overflow-y: auto; font-family: monospace; font-size: 0.9rem; line-height: 1.5; }
    .bot-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(90px, 1fr)); gap: 8px; }
    .bot-btn { text-align: center; padding: 10px 4px; }
    .bot-btn.selected { background: var(--accent); color: #111; }
    .btn-row { display: flex; gap: 8px; flex-wrap: wrap; }
    .play-btn {
      background: #81b64c !important;
      color: #111 !important;
      font-size: 1.15rem !important;
      padding: 14px 20px !important;
      font-weight: 700 !important;
      width: 100%;
      margin-top: 8px;
    }
    .thinking { color: #f1c40f !important; font-weight: 600; }
    .success { color: #81b64c !important; font-weight: 700; }
    .hidden { display: none !important; }
    footer { margin-top: 30px; text-align: center; color: var(--muted); font-size: 0.85rem; }
  </style>
</head>
<body>
<div class="container">
  <header>
    <h1>♟️ Chess App</h1>
    <div class="nav">
      <button id="btn-bots" class="primary">Gegen Bots</button>
      <button id="btn-both">Beide Farben</button>
      <button id="btn-puzzle">Rätsel</button>
    </div>
  </header>

  <div class="main">
    <div>
      <div id="board"></div>
      <div class="status" id="status">Wähle einen Bot und klicke auf „Partie starten“</div>
    </div>

    <div class="panel">
      <div class="clocks">
        <div class="clock" id="clock-black">5:00</div>
        <div class="clock" id="clock-white">5:00</div>
      </div>

      <div>
        <h3>Zeitkontrolle</h3>
        <select id="time-control">
          <option value="60">1 Minute</option>
          <option value="180">3 Minuten</option>
          <option value="300" selected>5 Minuten</option>
          <option value="600">10 Minuten</option>
        </select>
      </div>

      <!-- BOT BEREICH -->
      <div id="bot-section">
        <h3>Bot-Stärke wählen</h3>
        <div class="bot-grid" id="bot-grid"></div>

        <!-- DER WICHTIGE BUTTON -->
        <button id="btn-play" class="play-btn">▶ Partie starten</button>
      </div>

      <div class="btn-row">
        <button id="btn-hint" class="primary">💡 Bester Zug</button>
        <button id="btn-undo">↩ Zug zurück</button>
        <button id="btn-resign" class="danger">Aufgeben</button>
      </div>

      <div>
        <h3>Züge</h3>
        <div class="moves" id="moves"></div>
      </div>
    </div>
  </div>

  <footer>Partie starten funktioniert jetzt zuverlässig</footer>
</div>

<script>
  // ===================== VARIABLEN =====================
  const RATINGS = [100, 250, 350, 500, 600, 750, 850, 900, 1000, 1100, 1200, 1300];

  let game = new Chess();
  let board = null;
  let mode = "bot";
  let selectedRating = 500;
  let gameStarted = false;      // ← entscheidend
  let isPlayerTurn = true;
  let botThinking = false;
  let moveStack = [];
  let timeControl = 300;
  let whiteTime = 300;
  let blackTime = 300;
  let timerInterval = null;

  // ===================== BOARD =====================
  function initBoard() {
    board = Chessboard("board", {
      draggable: true,
      position: "start",
      pieceTheme: "https://chessboardjs.com/img/chesspieces/wikipedia/{piece}.png",
      onDragStart: onDragStart,
      onDrop: onDrop,
      onSnapEnd: () => board.position(game.fen())
    });
  }

  function onDragStart(source, piece) {
    // Partie muss gestartet sein
    if (!gameStarted) return false;
    if (game.game_over() || botThinking) return false;

    // Nur weiße Figuren erlauben (Spieler ist immer Weiß)
    if (piece.search(/^b/) !== -1) return false;
    if (!isPlayerTurn) return false;

    return true;
  }

  function onDrop(source, target) {
    if (!gameStarted || botThinking) return "snapback";

    const move = game.move({
      from: source,
      to: target,
      promotion: "q"
    });

    if (move === null) return "snapback";

    moveStack.push(move);
    updateMoves();
    updateStatus();

    if (!game.game_over()) {
      isPlayerTurn = false;
      // Bot antwortet nach kurzer Pause
      setTimeout(makeBotMove, 400);
    } else {
      onGameOver();
    }
  }

  // ===================== BOT =====================
  function makeBotMove() {
    if (game.game_over()) return;

    botThinking = true;
    updateStatus();

    setTimeout(() => {
      const depth = ratingToDepth(selectedRating);
      const best = getBestMove(depth);

      if (best) {
        const m = game.move(best);
        if (m) {
          moveStack.push(m);
          board.position(game.fen());
          updateMoves();
        }
      }

      botThinking = false;
      isPlayerTurn = true;
      updateStatus();

      if (game.game_over()) onGameOver();
    }, 100);
  }

  function ratingToDepth(r) {
    if (r <= 300) return 1;
    if (r <= 600) return 2;
    if (r <= 900) return 3;
    if (r <= 1100) return 4;
    return 5;
  }

  // ===================== EINFACHE ENGINE =====================
  function evaluate() {
    if (game.in_checkmate()) return game.turn() === "w" ? -99999 : 99999;
    if (game.in_draw()) return 0;

    const values = { p: 100, n: 320, b: 330, r: 500, q: 900, k: 20000 };
    let score = 0;
    const b = game.board();

    for (let r = 0; r < 8; r++) {
      for (let c = 0; c < 8; c++) {
        const p = b[r][c];
        if (p) {
          score += (p.color === "w" ? 1 : -1) * values[p.type];
        }
      }
    }
    return score;
  }

  function getBestMove(depth) {
    const moves = game.moves({ verbose: true });
    if (moves.length === 0) return null;

    let bestScore = -Infinity;
    let bestMove = moves[0];

    for (const m of moves) {
      game.move(m);
      const score = -minimax(depth - 1, -Infinity, Infinity);
      game.undo();

      if (score > bestScore) {
        bestScore = score;
        bestMove = m;
      }
    }
    return bestMove;
  }

  function minimax(depth, alpha, beta) {
    if (depth === 0 || game.game_over()) return evaluate();

    const moves = game.moves({ verbose: true });
    let maxEval = -Infinity;

    for (const m of moves) {
      game.move(m);
      const eval_ = -minimax(depth - 1, -beta, -alpha);
      game.undo();
      maxEval = Math.max(maxEval, eval_);
      alpha = Math.max(alpha, eval_);
      if (beta <= alpha) break;
    }
    return maxEval;
  }

  // ===================== PARTIE STARTEN (der wichtige Teil) =====================
  function startGame() {
    console.log("▶ Partie starten wurde geklickt");

    // Zustand zurücksetzen
    clearInterval(timerInterval);
    gameStarted = true;
    isPlayerTurn = true;
    botThinking = false;
    moveStack = [];

    game.reset();
    board.position("start");

    // Zeit
    timeControl = parseInt(document.getElementById("time-control").value) || 300;
    whiteTime = timeControl;
    blackTime = timeControl;
    startClock();

    updateMoves();
    updateStatus();

    document.getElementById("status").textContent = 
      "Partie gestartet! Du spielst Weiß. Ziehe eine Figur.";
    
    console.log("gameStarted ist jetzt:", gameStarted);
  }

  // Event-Listener für den Button
  document.getElementById("btn-play").addEventListener("click", startGame);

  // ===================== WEITERE FUNKTIONEN =====================
  function updateStatus() {
    const el = document.getElementById("status");

    if (!gameStarted) {
      el.textContent = `Bot ${selectedRating} ausgewählt – klicke auf „Partie starten“`;
    } else if (botThinking) {
      el.textContent = "Bot denkt…";
      el.className = "status thinking";
    } else if (game.in_checkmate()) {
      el.textContent = "Schachmatt! " + (game.turn() === "w" ? "Schwarz" : "Weiß") + " gewinnt.";
    } else if (game.in_draw()) {
      el.textContent = "Remis";
    } else if (game.in_check()) {
      el.textContent = "Schach!";
    } else {
      el.textContent = (game.turn() === "w" ? "Weiß" : "Schwarz") + " ist am Zug";
      el.className = "status";
    }

    document.getElementById("btn-undo").disabled = moveStack.length === 0 || botThinking || !gameStarted;
    document.getElementById("btn-hint").disabled = !gameStarted || botThinking;
  }

  function updateMoves() {
    const hist = game.history();
    let html = "";
    for (let i = 0; i < hist.length; i += 2) {
      html += `<div>${Math.floor(i/2)+1}. ${hist[i]} ${hist[i+1] || ""}</div>`;
    }
    document.getElementById("moves").innerHTML = html;
  }

  function startClock() {
    clearInterval(timerInterval);
    updateClocks();
    timerInterval = setInterval(() => {
      if (!gameStarted || game.game_over() || botThinking) return;
      if (game.turn() === "w") whiteTime--;
      else blackTime--;
      updateClocks();
      if (whiteTime <= 0 || blackTime <= 0) {
        clearInterval(timerInterval);
        alert(whiteTime <= 0 ? "Weiß hat die Zeit überschritten" : "Schwarz hat die Zeit überschritten");
        onGameOver();
      }
    }, 1000);
  }

  function updateClocks() {
    const fmt = t => `${Math.floor(t/60)}:${(t%60).toString().padStart(2,"0")}`;
    document.getElementById("clock-white").textContent = fmt(Math.max(0, whiteTime));
    document.getElementById("clock-black").textContent = fmt(Math.max(0, blackTime));
    document.getElementById("clock-white").classList.toggle("active", game.turn() === "w");
    document.getElementById("clock-black").classList.toggle("active", game.turn() === "b");
  }

  function onGameOver() {
    clearInterval(timerInterval);
    botThinking = false;
    updateStatus();
  }

  // Undo
  document.getElementById("btn-undo").addEventListener("click", () => {
    if (moveStack.length === 0 || botThinking || !gameStarted) return;

    // Zwei Züge zurück (Spieler + Bot)
    const count = Math.min(2, moveStack.length);
    for (let i = 0; i < count; i++) {
      game.undo();
      moveStack.pop();
    }
    isPlayerTurn = true;
    board.position(game.fen());
    updateMoves();
    updateStatus();
  });

  // Hinweis
  document.getElementById("btn-hint").addEventListener("click", () => {
    if (!gameStarted || botThinking) return;
    const best = getBestMove(3);
    if (best) {
      alert(`Bester Zug: ${best.san}\n${best.from} → ${best.to}`);
    }
  });

  // Aufgeben
  document.getElementById("btn-resign").addEventListener("click", () => {
    if (!gameStarted) return;
    if (confirm("Wirklich aufgeben?")) {
      onGameOver();
      document.getElementById("status").textContent = "Du hast aufgegeben.";
    }
  });

  // Bot-Auswahl
  function renderBots() {
    const grid = document.getElementById("bot-grid");
    grid.innerHTML = RATINGS.map(r => 
      `<button class="bot-btn ${r === selectedRating ? "selected" : ""}" data-r="${r}">${r}</button>`
    ).join("");

    grid.querySelectorAll(".bot-btn").forEach(btn => {
      btn.addEventListener("click", () => {
        selectedRating = parseInt(btn.dataset.r);
        renderBots();
        if (!gameStarted) updateStatus();
      });
    });
  }

  // Modus-Buttons
  document.getElementById("btn-bots").addEventListener("click", () => {
    mode = "bot";
    gameStarted = false;
    document.getElementById("bot-section").classList.remove("hidden");
    board.position("start");
    game.reset();
    moveStack = [];
    updateMoves();
    updateStatus();
  });

  document.getElementById("btn-both").addEventListener("click", () => {
    mode = "both";
    startGame(); // bei beiden Farben sofort starten
  });

  // ===================== START =====================
  renderBots();
  initBoard();
  updateStatus();
</script>
</body>
</html>
