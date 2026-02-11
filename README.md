<<!doctype html>
<html lang="no">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Ballong-skyting: Pro</title>
<style>
  :root { --bg: #111; --canvas-bg: #222; --panel: rgba(255,255,255,0.04); }
  html,body { height:100%; margin:0; overflow: hidden; }
  body {
    background:var(--bg);
    color:#fff;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    display:flex;
    align-items:start;
    justify-content:center;
    padding:20px;
  }
  .wrap {
    display:flex;
    flex-direction:column;
    gap:10px;
    align-items:center;
    width: 100%;
  }
  canvas {
    background:var(--canvas-bg);
    border:2px solid #555;
    cursor: crosshair;
    touch-action:none;
    width: 100%;
    max-width: 700px;
    height: auto;
    box-shadow: 0 10px 30px rgba(0,0,0,0.5);
  }
  .panel {
    background: var(--panel);
    padding:10px 20px;
    border-radius:8px;
    font-size:16px;
    display:flex;
    gap:20px;
    align-items:center;
    flex-wrap:wrap;
    border: 1px solid #444;
  }
  button {
    background:#3a7;
    color:#fff;
    border:none;
    padding:8px 15px;
    border-radius:4px;
    cursor:pointer;
    font-weight: bold;
  }
  button:hover { background: #4b8; }
  .note { color:#aaa; font-size:13px; }
</style>
</head>
<body>
<div class="wrap">
<div class="panel">
<div id="score">Score: 0</div>
<div id="lives">Liv: 3</div>
<div id="high">High: 0</div>
<button id="reset">Nullstill Highscore</button>
</div>
<canvas id="game" width="700" height="490" aria-label="Ballongspill"></canvas>
<div class="note">Skyt en ballong hvert 10. sekund for å holde livene! Farten øker hvert 30. sek.</div>
</div>
 
<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d', { alpha: false });
  let W = 700, H = 490;
  const scoreEl = document.getElementById('score');
  const highEl = document.getElementById('high');
  const livesEl = document.getElementById('lives');
  const resetBtn = document.getElementById('reset');
 
  // Lyd-motor (Web Audio API)
  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  function playPopSound() {
    if (audioCtx.state === 'suspended') audioCtx.resume();
    const osc = audioCtx.createOscillator();
    const gain = audioCtx.createGain();
    osc.type = 'sine';
    osc.frequency.setValueAtTime(150, audioCtx.currentTime);
    osc.frequency.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.2);
    gain.gain.setValueAtTime(0.3, audioCtx.currentTime);
    gain.gain.exponentialRampToValueAtTime(0.01, audioCtx.currentTime + 0.2);
    osc.connect(gain);
    gain.connect(audioCtx.destination);
    osc.start();
    osc.stop(audioCtx.currentTime + 0.2);
  }
 
  function resizeCanvas() {
    const rect = canvas.getBoundingClientRect();
    const dpr = Math.max(window.devicePixelRatio || 1, 1);
    W = Math.max(320, Math.floor(rect.width));
    H = Math.floor((W * 490) / 700);
    canvas.style.width = rect.width + 'px';
    canvas.style.height = H + 'px';
    canvas.width = Math.floor(W * dpr);
    canvas.height = Math.floor(H * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }
  window.addEventListener('resize', resizeCanvas);
  resizeCanvas();
 
  const BALLOON_COUNT = 7;
  const COLORS = ['#e74c3c','#ff6b6b','#ffd166','#6bcb77','#4d96ff','#c77df3'];
  let score = 0;
  let lives = 3;
  let high = parseInt(localStorage.getItem('ballong_high') || '0', 10);
  let gameStartTime = performance.now();
  let lastPopTimestamp = performance.now();
  let gameOver = false;
  let speedMultiplier = 1.0;
 
  highEl.textContent = 'High: ' + high;
 
  const balloons = [];
  function rand(min, max) { return Math.random() * (max - min) + min; }
 
  function spawnBalloon() {
    const r = rand(12, 28); // Variert størrelse
    const x = rand(r, W - r);
    const y = rand(r, H - r);
    const baseSpeed = rand(0.5, 1.5);
    const angle = rand(0, Math.PI * 2);
    return {
      x, y, r,
      vx: Math.cos(angle) * baseSpeed,
      vy: Math.sin(angle) * baseSpeed,
      color: COLORS[Math.floor(Math.random() * COLORS.length)],
      popping: false,
      popStart: 0
    };
  }
 
  function init() {
    balloons.length = 0;
    for (let i = 0; i < BALLOON_COUNT; i++) balloons.push(spawnBalloon());
    score = 0;
    lives = 3;
    speedMultiplier = 1.0;
    gameOver = false;
    gameStartTime = performance.now();
    lastPopTimestamp = performance.now();
    updateHUD();
  }
  init();
 
  function handlePointer(x, y) {
    if (gameOver) { init(); return; }
    
    for (let b of balloons) {
      if (b.popping) continue;
      const dx = x - b.x;
      const dy = y - b.y;
      if (Math.hypot(dx, dy) <= b.r) {
        b.popping = true;
        b.popStart = performance.now();
        
        // Poengberegning: mindre = mer poeng (fra 1 til 5 poeng)
        const points = Math.max(1, Math.round(35 / b.r));
        score += points;
        
        lastPopTimestamp = performance.now(); // Nullstill skuddklokke
        playPopSound();
        updateHUD();
        return;
      }
    }
  }
 
  canvas.addEventListener('pointerdown', e => {
    const rect = canvas.getBoundingClientRect();
    handlePointer(e.clientX - rect.left, e.clientY - rect.top);
  });
 
  function updateHUD() {
    scoreEl.textContent = 'Score: ' + score;
    livesEl.textContent = 'Liv: ' + lives;
    if (score > high) {
      high = score;
      localStorage.setItem('ballong_high', String(high));
      highEl.textContent = 'High: ' + high;
    }
    if (lives <= 0) gameOver = true;
  }
 
  function update(dt) {
    if (gameOver) return;
 
    const now = performance.now();
    
    // Øk tempo hvert 30. sekund
    const secondsElapsed = (now - gameStartTime) / 1000;
    speedMultiplier = 1.0 + Math.floor(secondsElapsed / 30) * 0.25;
 
    // Sjekk skuddklokke (10 sekunder)
    const timeSinceLastPop = (now - lastPopTimestamp) / 1000;
    if (timeSinceLastPop >= 10) {
      lives--;
      lastPopTimestamp = now; // Gi 10 nye sekunder
      updateHUD();
    }
 
    for (let b of balloons) {
      if (b.popping) {
        if (now - b.popStart > 250) Object.assign(b, spawnBalloon());
        continue;
      }
      // Bruk speedMultiplier her
      b.x += b.vx * dt * 0.06 * speedMultiplier;
      b.y += b.vy * dt * 0.06 * speedMultiplier;
 
      if (b.x < b.r || b.x > W - b.r) { b.vx *= -1; b.x = Math.max(b.r, Math.min(W-b.r, b.x)); }
      if (b.y < b.r || b.y > H - b.r) { b.vy *= -1; b.y = Math.max(b.r, Math.min(H-b.r, b.y)); }
    }
  }
 
  function draw() {
    ctx.fillStyle = '#222';
    ctx.fillRect(0,0,W,H);
 
    // Tegn skuddklokke-bar i bunnen
    const now = performance.now();
    const timeRatio = Math.max(0, 1 - (now - lastPopTimestamp) / 10000);
    ctx.fillStyle = timeRatio > 0.3 ? '#3a7' : '#e74c3c';
    ctx.fillRect(0, H - 5, W * timeRatio, 5);
 
    for (let b of balloons) {
      ctx.save();
      if (b.popping) {
        const t = (now - b.popStart) / 250;
        ctx.globalAlpha = 1 - t;
        ctx.translate(b.x, b.y);
        ctx.scale(1 + t, 1 + t);
        ctx.translate(-b.x, -b.y);
      }
      
      // Snor
      ctx.beginPath();
      ctx.strokeStyle = 'rgba(255,255,255,0.1)';
      ctx.moveTo(b.x, b.y + b.r);
      ctx.lineTo(b.x, b.y + b.r + 20);
      ctx.stroke();
 
      // Ballong
      ctx.beginPath();
      const g = ctx.createRadialGradient(b.x - b.r*0.3, b.y - b.r*0.3, b.r*0.1, b.x, b.y, b.r);
      g.addColorStop(0, '#fff8');
      g.addColorStop(0.3, b.color);
      g.addColorStop(1, '#000a');
      ctx.fillStyle = g;
      ctx.arc(b.x, b.y, b.r, 0, Math.PI*2);
      ctx.fill();
      ctx.restore();
    }
 
    // HUD på Canvas
    ctx.fillStyle = 'white';
    ctx.font = 'bold 14px Arial';
    ctx.fillText('TEMPO: x' + speedMultiplier.toFixed(1), 10, H - 20);
    ctx.textAlign = 'right';
    ctx.fillText('TID IGJEN: ' + Math.ceil(10 - (now - lastPopTimestamp)/1000) + 's', W - 10, H - 20);
    ctx.textAlign = 'left';
 
    if (gameOver) {
      ctx.fillStyle = 'rgba(0,0,0,0.8)';
      ctx.fillRect(0,0,W,H);
      ctx.fillStyle = 'white';
      ctx.font = 'bold 40px Arial';
      ctx.textAlign = 'center';
      ctx.fillText('GAME OVER', W/2, H/2);
      ctx.font = '20px Arial';
      ctx.fillText('Score: ' + score, W/2, H/2 + 40);
      ctx.fillText('Trykk for å prøve igjen', W/2, H/2 + 80);
    }
  }
 
  resetBtn.addEventListener('click', () => {
    localStorage.removeItem('ballong_high');
    high = 0;
    highEl.textContent = 'High: 0';
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
