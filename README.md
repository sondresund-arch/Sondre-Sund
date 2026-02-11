<!doctype html>
<html lang="no">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ballong-skyting: Pro</title>
<style>
  :root { --bg: #111; --canvas-bg: #222; --panel: rgba(255,255,255,0.04); --accent: #e74c3c; }
  html,body { height:100%; margin:0; overflow: hidden; }
  body {
    background:var(--bg);
    color:#fff;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    display:flex;
    align-items:center;
    justify-content:center;
    padding:10px;
  }
  .main-container {
    display: flex;
    gap: 20px;
    align-items: flex-start;
    max-width: 1000px;
    width: 100%;
  }
  .game-area {
    display:flex;
    flex-direction:column;
    gap:10px;
    flex-grow: 1;
  }
  canvas {
    background:var(--canvas-bg);
    border:2px solid #555;
    cursor: crosshair;
    touch-action:none;
    width: 100%;
    height: auto;
    box-shadow: 0 10px 30px rgba(0,0,0,0.5);
  }
  .side-panel {
    display: flex;
    flex-direction: column;
    gap: 15px;
    min-width: 150px;
  }
  .stat-box {
    background: var(--panel);
    padding: 15px;
    border-radius: 8px;
    border: 1px solid #444;
    text-align: center;
  }
  .timer-val {
    font-size: 32px;
    font-weight: bold;
    color: var(--accent);
    display: block;
  }
  .score-val { font-size: 24px; color: #3a7; font-weight: bold; }
  
  .top-bar {
    background: var(--panel);
    padding:10px;
    border-radius:8px;
    display:flex;
    justify-content: space-around;
    border: 1px solid #444;
  }
  button {
    background:#444;
    color:#fff;
    border:none;
    padding:8px;
    border-radius:4px;
    cursor:pointer;
    font-size: 11px;
  }
  button:hover { background: #555; }
 
  @media (max-width: 800px) {
    .main-container { flex-direction: column; align-items: center; }
    .side-panel { flex-direction: row; width: 100%; }
    .stat-box { flex: 1; padding: 5px; }
  }
</style>
</head>
<body>
 
<div class="main-container">
  <div class="game-area">
    <div class="top-bar">
      <div id="lives">Liv: 3</div>
      <div id="high">High: 0</div>
      <button id="reset">Nullstill Highscore</button>
    </div>
    <canvas id="game" width="700" height="490"></canvas>
  </div>
 
  <div class="side-panel">
    <div class="stat-box">
      <span>SKUDDKLOKKE</span>
      <span id="timer" class="timer-val">10.0</span>
    </div>
    <div class="stat-box">
      <span>SCORE</span>
      <div id="score" class="score-val">0</div>
    </div>
    <div style="font-size: 12px; color: #888; text-align: center;">
      Små ballonger = 3p<br>Store ballonger = 1p
    </div>
  </div>
</div>
 
<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d', { alpha: false });
  const scoreEl = document.getElementById('score');
  const highEl = document.getElementById('high');
  const livesEl = document.getElementById('lives');
  const timerEl = document.getElementById('timer');
  const resetBtn = document.getElementById('reset');
 
  let W = 700, H = 490;
  
  // Lyd-motor
  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  function playPopSound() {
    if (audioCtx.state === 'suspended') audioCtx.resume();
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.type = 'sine';
    osc.frequency.setValueAtTime(160, audioCtx.currentTime);
    osc.frequency.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.15);
    gain.gain.setValueAtTime(0.2, audioCtx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.15);
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    osc.start();
    osc.stop(audioCtx.currentTime + 0.15);
  }
 
  function resizeCanvas() {
    const rect = canvas.getBoundingClientRect();
    const dpr = window.devicePixelRatio || 1;
    W = 700; H = 490; // Logisk størrelse
    canvas.width = W * dpr;
    canvas.height = H * dpr;
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }
  window.addEventListener('resize', resizeCanvas);
  resizeCanvas();
 
  const BALLOON_COUNT = 6;
  const COLORS = ['#e74c3c','#ff6b6b','#ffd166','#6bcb77','#4d96ff','#c77df3'];
  let score = 0, lives = 3, gameOver = false;
  let high = parseInt(localStorage.getItem('ballong_high_v2') || '0', 10);
  let gameStartTime = performance.now();
  let lastPopTimestamp = performance.now();
  let speedMultiplier = 1.0;
 
  highEl.textContent = 'High: ' + high;
 
  const balloons = [];
  function spawnBalloon() {
    const r = Math.random() * (28 - 12) + 12; // Radius mellom 12 og 28
    return {
      x: Math.random() * (W - 60) + 30,
      y: Math.random() * (H - 60) + 30,
      r: r,
      vx: (Math.random() - 0.5) * 2.5,
      vy: (Math.random() - 0.5) * 2.5,
      color: COLORS[Math.floor(Math.random() * COLORS.length)],
      popping: false,
      popStart: 0
    };
  }
 
  function init() {
    balloons.length = 0;
    for (let i = 0; i < BALLOON_COUNT; i++) balloons.push(spawnBalloon());
    score = 0; lives = 3; speedMultiplier = 1.0; gameOver = false;
    gameStartTime = performance.now();
    lastPopTimestamp = performance.now();
    updateHUD();
  }
  init();
 
  canvas.addEventListener('pointerdown', e => {
    if (gameOver) { init(); return; }
    const rect = canvas.getBoundingClientRect();
    const scaleX = W / rect.width;
    const scaleY = H / rect.height;
    const mx = (e.clientX - rect.left) * scaleX;
    const my = (e.clientY - rect.top) * scaleY;
 
    for (let b of balloons) {
      if (!b.popping && Math.hypot(mx - b.x, my - b.y) <= b.r) {
        b.popping = true;
        b.popStart = performance.now();
        
        // POENGLOGIKK: De minste (r < 16) gir 3 poeng, resten 1.
        score += (b.r < 16) ? 3 : 1;
        
        lastPopTimestamp = performance.now(); // RESET KLOKKE
        playPopSound();
        updateHUD();
        break;
      }
    }
  });
 
  function updateHUD() {
    scoreEl.textContent = score;
    livesEl.textContent = 'Liv: ' + lives;
    if (score > high) {
      high = score;
      localStorage.setItem('ballong_high_v2', String(high));
      highEl.textContent = 'High: ' + high;
    }
    if (lives <= 0) gameOver = true;
  }
 
  function update(dt) {
    if (gameOver) return;
    const now = performance.now();
    
    // Øk tempo hvert 30. sekund
    speedMultiplier = 1.0 + Math.floor((now - gameStartTime) / 30000) * 0.3;
 
    // Skuddklokke logikk
    let timeRemaining = Math.max(0, 10 - (now - lastPopTimestamp) / 1000);
    timerEl.textContent = timeRemaining.toFixed(1);
    
    if (timeRemaining <= 0) {
      lives--;
      lastPopTimestamp = now;
      updateHUD();
    }
 
    for (let b of balloons) {
      if (b.popping) {
        if (now - b.popStart > 250) Object.assign(b, spawnBalloon());
        continue;
      }
      b.x += b.vx * dt * 0.06 * speedMultiplier;
      b.y += b.vy * dt * 0.06 * speedMultiplier;
      if (b.x < b.r || b.x > W - b.r) b.vx *= -1;
      if (b.y < b.r || b.y > H - b.r) b.vy *= -1;
    }
  }
 
  function draw() {
    ctx.fillStyle = '#222';
    ctx.fillRect(0,0,W,H);
 
    for (let b of balloons) {
      ctx.save();
      if (b.popping) {
        const t = (performance.now() - b.popStart) / 250;
        ctx.globalAlpha = 1 - t;
        ctx.translate(b.x, b.y);
        ctx.scale(1 + t, 1 + t);
        ctx.translate(-b.x, -b.y);
      }
      
      // Ballong-kropp
      ctx.beginPath();
      const g = ctx.createRadialGradient(b.x - b.r*0.3, b.y - b.r*0.3, b.r*0.1, b.x, b.y, b.r);
      g.addColorStop(0, '#fff6');
      g.addColorStop(0.4, b.color);
      g.addColorStop(1, '#0004');
      ctx.fillStyle = g;
      ctx.arc(b.x, b.y, b.r, 0, Math.PI*2);
      ctx.fill();
      ctx.restore();
    }
 
    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.85)';
      ctx.fillRect(0,0,W,H);
      ctx.fillStyle = 'white';
      ctx.textAlign = 'center';
      ctx.font = 'bold 40px Arial';
      ctx.fillText('GAME OVER', W/2, H/2 - 20);
      ctx.font = '20px Arial';
      ctx.fillText('Score: ' + score, W/2, H/2 + 20);
      ctx.fillText('Trykk for å starte på nytt', W/2, H/2 + 60);
    }
  }
 
  resetBtn.addEventListener('click', () => {
    localStorage.removeItem('ballong_high_v2');
    high = 0; highEl.textContent = 'High: 0';
  });
 
  let last = performance.now();
  function loop(ts) {
    update(ts - last);
    draw();
    last = ts;
    requestAnimationFrame(loop);
  }
  requestAnimationFrame(loop);
})();
</script>
</body>
</html>
