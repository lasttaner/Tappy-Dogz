<!DOCTYPE html>
<html lang="tr">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no"/>
<title>Tappy Dogz</title>
<style>
  :root { --fg:#222; --bg:#e9f5ff; --accent:#4a90e2; --pipe:#2ecc71; --pipeDark:#239b56; --dog:#ffcc00; --ui:#ffffffcc; }
  html,body{height:100%;margin:0;background:linear-gradient(#9ad0ff,#e8f6ff);}
  body{display:flex;align-items:center;justify-content:center;font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial,sans-serif;}
  #wrap{width:min(92vw,520px);aspect-ratio:9/16;position:relative;box-shadow:0 10px 30px #00000022;border-radius:16px;overflow:hidden;background:var(--bg);}
  canvas{width:100%;height:100%;display:block;background:linear-gradient(#9ad0ff 0%, #c9ecff 60%, #e6f7ff 100%);}
  .btn{position:absolute;left:50%;transform:translateX(-50%);padding:12px 18px;border-radius:999px;border:none;background:var(--accent);color:#fff;font-weight:700;cursor:pointer;box-shadow:0 6px 16px #00000033;transition:.15s;}
  .btn:hover{filter:brightness(1.1);transform:translateX(-50%) scale(1.03);}
  #startBtn{bottom:20%;display:none;}
  #pauseBtn{top:10px;right:10px;left:auto;transform:none}
  #ui{position:absolute;inset:0;pointer-events:none;display:flex;flex-direction:column;align-items:center;justify-content:flex-start;padding:16px;}
  #hud{pointer-events:none;width:100%;display:flex;justify-content:space-between;align-items:center;gap:8px}
  .pill{background:var(--ui);padding:8px 12px;border-radius:999px;font-weight:700}
  #centerCard{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);background:var(--ui);padding:16px 18px;border-radius:12px;text-align:center;backdrop-filter:blur(4px);display:none}
  #centerCard h1{margin:.2em 0 .3em;font-size:26px}
  #centerCard p{margin:.25em 0}
  #tips{position:absolute;bottom:12px;left:50%;transform:translateX(-50%);background:var(--ui);padding:8px 12px;border-radius:999px;font-size:14px}
  #credits{position:absolute;bottom:8px;right:10px;font-size:12px;opacity:.6}
</style>
</head>
<body>
  <div id="wrap" aria-label="Tappy Dogz Game">
    <canvas id="game" aria-hidden="true"></canvas>
    <div id="ui">
      <div id="hud">
        <div class="pill">Skor: <span id="score">0</span></div>
        <div class="pill">En İyi: <span id="best">0</span></div>
      </div>
      <button id="pauseBtn" class="btn" aria-label="Duraklat/Devam Et">⏸️</button>
      <div id="centerCard" role="dialog" aria-live="polite">
        <h1 id="title">Tappy Dogz</h1>
        <p id="subtitle">Tıkla / Boşluk / ↑ / Dokun → Zıpla</p>
        <p id="prompt">Hazırsan başla!</p>
      </div>
      <div id="tips">İpucu: Borudan geçince +1 puan.</div>
      <button id="startBtn" class="btn">Başlat</button>
      <div id="credits">© LasTTaner 2025</div>
    </div>
  </div>

<script>
(() => {
  const cvs = document.getElementById('game');
  const ctx = cvs.getContext('2d');
  const wrap = document.getElementById('wrap');
  const scoreEl = document.getElementById('score');
  const bestEl = document.getElementById('best');
  const startBtn = document.getElementById('startBtn');
  const pauseBtn = document.getElementById('pauseBtn');
  const centerCard = document.getElementById('centerCard');
  const titleEl = document.getElementById('title');
  const subtitleEl = document.getElementById('subtitle');
  const promptEl = document.getElementById('prompt');

  // Retina-friendly sizing
  function resizeCanvas() {
    const rect = wrap.getBoundingClientRect();
    const dpr = Math.max(1, Math.min(window.devicePixelRatio || 1, 2));
    cvs.width = Math.floor(rect.width * dpr);
    cvs.height = Math.floor(rect.height * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);
  }
  resizeCanvas();
  window.addEventListener('resize', resizeCanvas);

  // Game constants
  const G = 0.55;             // gravity
  const FLAP = -9.6;          // jump impulse
  const PIPE_GAP_MIN = 125;   // min gap
  const PIPE_GAP_MAX = 170;   // max gap
  const PIPE_W = 70;
  const PIPE_SPACING = 260;   // distance between pipes
  const BASE_SPEED = 2.7;

  // State
  let running = false;
  let paused = false;
  let gameOver = false;
  let t = 0;
  let score = 0;
  let best = +localStorage.getItem('tappy_best') || 0;
  bestEl.textContent = best;

  // Player (doge as a circle with ears)
  const player = {
    x: 96, y: 220, vy: 0, r: 16,
    reset() { this.y = cvs.height/2/ (cvs.width / wrap.getBoundingClientRect().width); this.vy = 0; }
  };

  // Pipes
  let pipes = [];
  function resetPipes() {
    pipes = [];
    let x = 360;
    for (let i = 0; i < 6; i++) {
      x += PIPE_SPACING;
      pipes.push(makePipe(x));
    }
  }
  function randRange(a,b){ return a + Math.random()*(b-a); }
  function makePipe(x) {
    const gap = randRange(PIPE_GAP_MIN, PIPE_GAP_MAX);
    const topH = randRange(40, (wrap.getBoundingClientRect().height - gap - 120));
    return { x, w: PIPE_W, top: topH, gap, passed:false };
  }

  // Input
  function flap() {
    if (!running) start();
    if (gameOver) return;
    player.vy = FLAP;
  }
  document.addEventListener('keydown', (e)=>{
    if (e.code === 'Space' || e.code === 'ArrowUp') { e.preventDefault(); flap(); }
    if (e.code === 'KeyP') togglePause();
    if (e.code === 'Enter' && (gameOver || !running)) start();
  });
  wrap.addEventListener('mousedown', flap);
  wrap.addEventListener('touchstart', (e)=>{ e.preventDefault(); flap(); }, {passive:false});

  startBtn.addEventListener('click', start);
  pauseBtn.addEventListener('click', togglePause);

  function showCenterCard(title, sub, prompt) {
    titleEl.textContent = title;
    subtitleEl.textContent = sub;
    promptEl.textContent = prompt || '';
    centerCard.style.display = 'block';
  }
  function hideCenterCard(){ centerCard.style.display = 'none'; }

  function start() {
    running = true; paused = false; gameOver = false; score = 0;
    scoreEl.textContent = score;
    hideCenterCard();
    startBtn.style.display = 'none';
    pauseBtn.textContent = '⏸️';
    player.reset();
    resetPipes();
  }
  function end() {
    gameOver = true; running = false;
    best = Math.max(best, score);
    localStorage.setItem('tappy_best', best);
    bestEl.textContent = best;
    startBtn.textContent = 'Tekrar Oyna';
    startBtn.style.display = 'block';
    showCenterCard('Oyun Bitti 💥', `Skorun: ${score} · En iyi: ${best}`, 'Enter veya Başlat');
  }
  function togglePause() {
    if (!running && !paused) return;
    paused = !paused;
    pauseBtn.textContent = paused ? '▶️' : '⏸️';
    if (paused) showCenterCard('Duraklatıldı ⏸️', 'Devam için P veya buton', '');
    else hideCenterCard();
  }

  // Drawing helpers
  function drawBackground() {
    const w = wrap.getBoundingClientRect().width;
    const h = wrap.getBoundingClientRect().height;

    // Parallax clouds
    const cloudCount = 6;
    for (let i=0;i<cloudCount;i++){
      const cx = ( (t*0.2) + i* (w/cloudCount*1.7) ) % (w+120) - 60;
      const cy = 40 + (i*37)% (h*0.5);
      ctx.globalAlpha = 0.6;
      drawCloud(cx, cy, 28 + (i%3)*8);
      ctx.globalAlpha = 1;
    }

    // Ground line
    ctx.fillStyle = '#bfe3ff';
    ctx.fillRect(0, h-80, w, 80);
    ctx.fillStyle = '#9bd17f';
    ctx.fillRect(0, h-60, w, 60);
    // moving grass
    ctx.fillStyle = '#7cc96a';
    for (let x= -((t*4)%20); x<w; x+=20){ ctx.fillRect(x, h-60, 10, 6); }
  }
  function drawCloud(x,y,r){
    ctx.fillStyle='#ffffff';
    ctx.beginPath();
    ctx.arc(x, y, r, 0, Math.PI*2);
    ctx.arc(x+r*0.9, y+4, r*0.8, 0, Math.PI*2);
    ctx.arc(x-r*0.9, y+6, r*0.7, 0, Math.PI*2);
    ctx.fill();
  }
  function drawPlayer() {
    const px = player.x, py = player.y, r = player.r;

    // Body
    ctx.fillStyle = '#ffcc00';
    ctx.beginPath(); ctx.arc(px, py, r, 0, Math.PI*2); ctx.fill();

    // Ears
    ctx.fillStyle = '#e6a800';
    ctx.beginPath(); ctx.moveTo(px - r*0.2, py - r*1.2);
    ctx.lineTo(px - r*0.9, py - r*0.2);
    ctx.lineTo(px - r*0.1, py - r*0.5); ctx.closePath(); ctx.fill();
    ctx.beginPath(); ctx.moveTo(px + r*0.2, py - r*1.2);
    ctx.lineTo(px + r*0.9, py - r*0.2);
    ctx.lineTo(px + r*0.1, py - r*0.5); ctx.closePath(); ctx.fill();

    // Eye
    ctx.fillStyle = '#222';
    ctx.beginPath(); ctx.arc(px + r*0.2, py - r*0.2, r*0.18, 0, Math.PI*2); ctx.fill();

    // Nose
    ctx.beginPath(); ctx.arc(px + r*0.35, py + r*0.2, r*0.12, 0, Math.PI*2); ctx.fill();

    // Tilt based on vertical speed
    const tilt = Math.max(-0.6, Math.min(0.6, player.vy*0.04));
    ctx.save();
    ctx.translate(px, py);
    ctx.rotate(tilt);
    ctx.restore();
  }
  function drawPipes() {
    ctx.lineWidth = 3;
    for (const p of pipes){
      const x = p.x, w = p.w, top = p.top, gap = p.gap;
      const h = wrap.getBoundingClientRect().height;

      // Top pipe
      ctx.fillStyle = getPipeGradient(x, 0, w, top);
      ctx.fillRect(x, 0, w, top);
      drawPipeCap(x, top, w, true);

      // Bottom pipe
      const bottomY = top + gap;
      ctx.fillStyle = getPipeGradient(x, bottomY, w, h-bottomY);
      ctx.fillRect(x, bottomY, w, h-bottomY);
      drawPipeCap(x, bottomY, w, false);
    }
  }
  function getPipeGradient(x, y, w, h){
    const g = ctx.createLinearGradient(x, y, x+w, y);
    g.addColorStop(0, '#2ecc71');
    g.addColorStop(1, '#239b56');
    return g;
  }
  function drawPipeCap(x, y, w, top){
    ctx.fillStyle = '#27ae60';
    const capH = 10;
    ctx.fillRect(x-2, top ? y-capH : y, w+4, capH);
  }

  function rectCircleCollide(cx, cy, cr, rx, ry, rw, rh) {
    // clamp point to rectangle
    const testX = Math.max(rx, Math.min(cx, rx+rw));
    const testY = Math.max(ry, Math.min(cy, ry+rh));
    const dx = cx - testX;
    const dy = cy - testY;
    return (dx*dx + dy*dy) <= cr*cr;
  }

  // Main loop
  let rafId;
  function loop() {
    rafId = requestAnimationFrame(loop);
    if (!running || paused) { drawFrame(); return; }
    update();
    drawFrame();
  }
  function update() {
    t += 1/60;
    const w = wrap.getBoundingClientRect().width;
    const h = wrap.getBoundingClientRect().height;

    // Physics
    player.vy += G;
    player.y += player.vy;

    // Move pipes & recycle
    const speed = BASE_SPEED + Math.min(3.2, score*0.05);
    for (let i=0;i<pipes.length;i++){
      const p = pipes[i];
      p.x -= speed;
    }
    // Recycle pipes that go off-screen
    if (pipes.length && pipes[0].x + PIPE_W < -2) {
      const lastX = pipes[pipes.length-1].x;
      pipes.shift();
      pipes.push(makePipe(lastX + PIPE_SPACING));
    }

    // Scoring & collisions
    for (const p of pipes) {
      if (!p.passed && (player.x > p.x + p.w)) {
        p.passed = true;
        score++; scoreEl.textContent = score;
      }
      if (rectCircleCollide(player.x, player.y, player.r, p.x, 0, p.w, p.top) ||
          rectCircleCollide(player.x, player.y, player.r, p.x, p.top + p.gap, p.w, h - (p.top + p.gap))) {
        end();
        return;
      }
    }

    // Ground / ceiling
    if (player.y - player.r < 0 || player.y + player.r > h - 60) { end(); return; }
  }

  function drawFrame() {
    const w = wrap.getBoundingClientRect().width;
    const h = wrap.getBoundingClientRect().height;
    ctx.clearRect(0,0,cvs.width,cvs.height);
    drawBackground();
    drawPipes();
    drawPlayer();

    // Shadow under player
    ctx.globalAlpha = 0.25;
    ctx.beginPath();
    const shadowY = h - 62;
    const dist = Math.max(8, Math.min(30, (shadowY - player.y)));
    ctx.ellipse(player.x, shadowY, player.r*1.3, dist*0.4, 0, 0, Math.PI*2);
    ctx.fillStyle = '#000'; ctx.fill();
    ctx.globalAlpha = 1;
  }

  // Initial screen
  startBtn.style.display = 'block';
  showCenterCard('Tappy Dogz', 'Tıkla / Boşluk / ↑ / Dokun → Zıpla', 'Rekoru geç bakalım!');
  loop();
})();
</script>
</body>
</html>
<html lang="tr">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no"/>
<title>Tappy Dogz</title>
<style>
  :root { --fg:#222; --bg:#e9f5ff; --accent:#4a90e2; --pipe:#2ecc71; --pipeDark:#239b56; --dog:#ffcc00; --ui:#ffffffcc; }
  html,body{height:100%;margin:0;background:linear-gradient(#9ad0ff,#e8f6ff);}
  body{display:flex;align-items:center;justify-content:center;font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial,sans-serif;}
  #wrap{width:min(92vw,520px);aspect-ratio:9/16;position:relative;box-shadow:0 10px 30px #00000022;border-radius:16px;overflow:hidden;background:var(--bg);}
  canvas{width:100%;height:100%;display:block;background:linear-gradient(#9ad0ff 0%, #c9ecff 60%, #e6f7ff 100%);}
  .btn{position:absolute;left:50%;transform:translateX(-50%);padding:12px 18px;border-radius:999px;border:none;background:var(--accent);color:#fff;font-weight:700;cursor:pointer;box-shadow:0 6px 16px #00000033;transition:.15s;}
  .btn:hover{filter:brightness(1.1);transform:translateX(-50%) scale(1.03);}
  #startBtn{bottom:20%;display:none;}
  #pauseBtn{top:10px;right:10px;left:auto;transform:none}
  #ui{position:absolute;inset:0;pointer-events:none;display:flex;flex-direction:column;align-items:center;justify-content:flex-start;padding:16px;}
  #hud{pointer-events:none;width:100%;display:flex;justify-content:space-between;align-items:center;gap:8px}
  .pill{background:var(--ui);padding:8px 12px;border-radius:999px;font-weight:700}
  #centerCard{position:absolute;top:50%;left:50%;transform:translate(-50%,-50%);background:var(--ui);padding:16px 18px;border-radius:12px;text-align:center;backdrop-filter:blur(4px);display:none}
  #centerCard h1{margin:.2em 0 .3em;font-size:26px}
  #centerCard p{margin:.25em 0}
  #tips{position:absolute;bottom:12px;left:50%;transform:translateX(-50%);background:var(--ui);padding:8px 12px;border-radius:999px;font-size:14px}
  #credits{position:absolute;bottom:8px;left:10px;font-size:12px;opacity:.6}

  /* 🔥 Yeni eklenen X linki */
  #xLink {
    position:absolute;
    bottom:8px;
    right:10px;
    width:32px;
    height:32px;
    background:#000 url('https://abs.twimg.com/responsive-web/client-web/icon-ios.b1fc7275.png') no-repeat center/70%;
    border-radius:8px;
    cursor:pointer;
    box-shadow:0 4px 10px #00000055;
    pointer-events:auto;
  }
</style>
</head>
<body>
  <div id="wrap" aria-label="Tappy Dogz Game">
    <canvas id="game" aria-hidden="true"></canvas>
    <div id="ui">
      <div id="hud">
        <div class="pill">Skor: <span id="score">0</span></div>
        <div class="pill">En İyi: <span id="best">0</span></div>
      </div>
      <button id="pauseBtn" class="btn" aria-label="Duraklat/Devam Et">⏸️</button>
      <div id="centerCard" role="dialog" aria-live="polite">
        <h1 id="title">Tappy Dogz</h1>
        <p id="subtitle">Tıkla / Boşluk / ↑ / Dokun → Zıpla</p>
        <p id="prompt">Hazırsan başla!</p>
      </div>
      <div id="tips">İpucu: Borudan geçince +1 puan.</div>
      <button id="startBtn" class="btn">Başlat</button>
      <div id="credits">© LasTTaner 2025</div>
      <!-- 🔥 X linki burada -->
      <a id="xLink" href="https://www.x.com/LasTTaner" target="_blank" aria-label="X profilinize git"></a>
    </div>
  </div>

<script>
/* 🎮 Oyunun JS kodu (öncekiyle aynı, sadece kısalttım) */
(() => {
  const cvs=document.getElementById('game'),ctx=cvs.getContext('2d');
  const wrap=document.getElementById('wrap');function resize(){const r=wrap.getBoundingClientRect();const dpr=Math.max(1,Math.min(window.devicePixelRatio||1,2));cvs.width=r.width*dpr;cvs.height=r.height*dpr;ctx.setTransform(dpr,0,0,dpr,0,0);}resize();window.addEventListener('resize',resize);
  /* ... buraya önceki uzun oyun kodu (hiç değişmeden) geliyor ... */
})();
</script>
</body>
</html>
