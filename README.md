<!DOCTYPE html>
<html lang="uz">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Hard Table Tennis (Pong)</title>
  <style>
    :root {
      --bg: #0b1020;
      --fg: #e2e8f0;
      --muted: #94a3b8;
      --accent: #22d3ee;
      --danger: #f43f5e;
      --good: #22c55e;
    }
    html, body { height: 100%; }
    body {
      margin: 0;
      background: radial-gradient(1000px 600px at 70% 20%, #0e1630, var(--bg));
      color: var(--fg);
      font-family: system-ui, -apple-system, Segoe UI, Roboto, Ubuntu, Cantarell, 'Noto Sans', 'Helvetica Neue', Arial, 'Apple Color Emoji', 'Segoe UI Emoji';
      overflow: hidden;
      user-select: none;
    }

    #wrap { position: relative; height: 100vh; width: 100vw; }
    canvas { display: block; width: 100%; height: 100%; }
    .hud {
      position: absolute; inset: 0; pointer-events: none; display: grid; grid-template-rows: auto 1fr auto; padding: 12px; }
    .row { display:flex; align-items:center; justify-content:space-between; gap:12px; }
    .badge { font-weight:700; letter-spacing:.08em; font-size:12px; color:var(--muted); }
    .score { font-variant-numeric: tabular-nums; font-weight:800; font-size:min(8vh,72px); line-height:1; }
    .centerMsg { pointer-events:none; position:absolute; inset:0; display:grid; place-items:center; text-align:center; }
    .centerMsg h1 { margin:0; font-size:min(6vh,56px); text-shadow:0 4px 24px rgba(34,211,238,.35); }
    .centerMsg p { margin:.25rem 0 0; color:var(--muted); }
    .controls { font-size:12px; color:var(--muted); }
    .btnbar { position:absolute; right:12px; bottom:12px; display:flex; gap:8px; }
    .btn { pointer-events:auto; border:1px solid #1f2937; background:#0f172a; color:#e5e7eb; padding:8px 10px; border-radius:12px; font-weight:600; font-size:12px; text-transform:uppercase; letter-spacing:.06em; box-shadow:0 6px 18px rgba(2,6,23,.4) inset, 0 4px 16px rgba(15,23,42,.4); }
    .btn:active { transform: translateY(1px); }
  </style>
</head>
<body>
  <div id="wrap">
    <canvas id="game" aria-label="Pong game" role="img"></canvas>
    <div class="hud">
      <div class="row">
        <div class="badge">HARD MODE • spin + AI prediction</div>
        <div class="score" id="score">0 : 0</div>
        <div class="badge" id="fps">FPS</div>
      </div>
      <div class="row" style="justify-content:center;">
        <div class="controls">Controls: Mouse / Touch to move • W/S ↑/↓ • P = Pause • R = Restart</div>
      </div>
    </div>
    <div class="btnbar">
      <button class="btn" id="pauseBtn">Pause</button>
      <button class="btn" id="restartBtn">Restart</button>
    </div>
    <div class="centerMsg" id="center"></div>
  </div>

  <script>

  const clamp = (v, a, b) => Math.max(a, Math.min(b, v));
  const rand = (a, b) => a + Math.random() * (b - a);


  const audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  function beep(freq=440, dur=0.06, type='sine', gain=0.02) {
    const o = audioCtx.createOscillator();
    const g = audioCtx.createGain();
    o.type = type; o.frequency.value = freq; g.gain.value = gain;
    o.connect(g); g.connect(audioCtx.destination);
    o.start(); o.stop(audioCtx.currentTime + dur);
  }


  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  const hudScore = document.getElementById('score');
  const center = document.getElementById('center');
  const fpsLabel = document.getElementById('fps');
  const pauseBtn = document.getElementById('pauseBtn');
  const restartBtn = document.getElementById('restartBtn');

  let W = 0, H = 0, DPR = Math.min(2, window.devicePixelRatio || 1);

  const STATE = {
    running: false,
    paused: false,
    countdown: 0,
    serving: 1, // 1 = player (right), -1 = AI (left)
    toWin: 11,
    winBy: 2,
    player: { y: 0, vy: 0, score: 0 },
    ai: { y: 0, vy: 0, score: 0, missFactor: 0.2 },
    paddle: { w: 14, h: 110, margin: 22 },
    ball: { x: 0, y: 0, r: 8, vx: 0, vy: 0, speed: 560, max: 1420 },
    particles: [],
    lastTime: 0,
    fps: 0
  };

  function resize() {
    const rect = canvas.getBoundingClientRect();
    W = Math.floor(rect.width * DPR);
    H = Math.floor(rect.height * DPR);
    canvas.width = W; canvas.height = H; ctx.setTransform(DPR,0,0,DPR,0,0);
    // scale paddle size with height
    STATE.paddle.h = Math.max(70, Math.round(H * 0.16));
  }
  window.addEventListener('resize', resize);
  resize();

  function drawCourt() {
    ctx.clearRect(0,0,canvas.width,canvas.height);
    // Table gradient
    const g = ctx.createLinearGradient(0,0,W,0);
    g.addColorStop(0,'#0b122a'); g.addColorStop(1,'#0a1933');
    ctx.fillStyle = g; ctx.fillRect(0,0,W,H);
    // Net
    ctx.strokeStyle = 'rgba(255,255,255,.22)';
    ctx.lineWidth = 3; ctx.setLineDash([12,14]);
    ctx.beginPath(); ctx.moveTo(W/2, 18); ctx.lineTo(W/2, H-18); ctx.stroke();
    ctx.setLineDash([]);
    // Glow edges
    ctx.shadowBlur = 30; ctx.shadowColor = 'rgba(34,211,238,.25)';
    ctx.strokeStyle = 'rgba(34,211,238,.25)'; ctx.strokeRect(10,10,W-20,H-20);
    ctx.shadowBlur = 0;
  }

  function drawPaddle(x, y, isPlayer) {
    const r = 10; const w = STATE.paddle.w; const h = STATE.paddle.h;
    ctx.fillStyle = isPlayer ? '#22d3ee' : '#94a3b8';
    ctx.beginPath();
    ctx.moveTo(x, y - h/2 + r);
    ctx.arcTo(x, y - h/2, x + w, y - h/2, r);
    ctx.arcTo(x + w, y - h/2, x + w, y - h/2 + r, r);
    ctx.arcTo(x + w, y + h/2, x, y + h/2, r);
    ctx.arcTo(x, y + h/2, x, y - h/2, r);
    ctx.closePath(); ctx.fill();
  }

  function drawBall() {
    const b = STATE.ball;
    const grd = ctx.createRadialGradient(b.x+2, b.y-2, 2, b.x, b.y, b.r*2);
    grd.addColorStop(0,'#fff'); grd.addColorStop(1,'#a3e7ff');
    ctx.fillStyle = grd;
    ctx.beginPath(); ctx.arc(b.x, b.y, b.r, 0, Math.PI*2); ctx.fill();
  }

  function spawnParticles(x,y,color,count=12) {
    for (let i=0;i<count;i++) {
      STATE.particles.push({
        x, y,
        vx: rand(-240,240),
        vy: rand(-240,240),
        life: rand(.18,.35),
        t: 0,
        color
      });
    }
  }

  function stepParticles(dt) {
    const arr = STATE.particles; ctx.save();
    for (let i=arr.length-1; i>=0; i--) {
      const p = arr[i]; p.t += dt; if (p.t >= p.life) { arr.splice(i,1); continue; }
      const k = 1 - p.t/p.life; // fade
      p.x += p.vx*dt; p.y += p.vy*dt; p.vy += 900*dt; // gravity for flair
      ctx.globalAlpha = Math.max(0, k*.9);
      ctx.fillStyle = p.color;
      ctx.fillRect(p.x, p.y, 3, 3);
    }
    ctx.globalAlpha = 1; ctx.restore();
  }

  function resetBall(serveDir=1) {
    const b = STATE.ball;
    b.x = W/2; b.y = H/2;
    // random serve angle with limits to avoid too flat
    const angle = rand(-0.7, 0.7);
    b.speed = rand(520, 620) * (1 + 0.08 * (STATE.player.score + STATE.ai.score));
    b.vx = Math.cos(angle) * b.speed * serveDir;
    b.vy = Math.sin(angle) * b.speed;
  }

  function startCountdown() {
    STATE.countdown = 3; center.innerHTML = `<h1>3</h1>`;
    const tick = () => {
      if (STATE.paused) return; // wait if paused
      if (STATE.countdown > 1) {
        STATE.countdown--; center.innerHTML = `<h1>${STATE.countdown}</h1>`; beep(740, .08, 'square', .03);
        setTimeout(tick, 600);
      } else {
        STATE.countdown = 0; center.innerHTML = '';
        resetBall(STATE.serving);
        STATE.running = true; beep(880, .06, 'triangle', .02);
      }
    }
    setTimeout(tick, 600);
  }

  function restart() {
    STATE.player.score = 0; STATE.ai.score = 0; updateScore();
    STATE.serving = Math.random() > .5 ? 1 : -1;
    STATE.ai.missFactor = 0.25; // easier at first
    STATE.running = false; STATE.paused = false; center.innerHTML = '';
    startCountdown();
  }

  function updateScore() {
    hudScore.textContent = `${STATE.ai.score} : ${STATE.player.score}`;
  }

  function winCheck() {
    const {toWin, winBy, player, ai} = STATE;
    const lead = Math.abs(player.score - ai.score);
    const maxScore = Math.max(player.score, ai.score);
    if (maxScore >= toWin && lead >= winBy) {
      STATE.running = false;
      center.innerHTML = `<h1>${player.score > ai.score ? 'YOU WIN 🏆' : 'AI WINS 💀'}</h1><p>Press R to play again</p>`;
      beep(player.score>ai.score? 1320:280, .25, 'sawtooth', .03);
      return true;
    }
    return false;
  }

  // ====== Input ======
  let targetY = H/2;
  function setTargetFromY(y) { targetY = clamp(y, STATE.paddle.h/2 + 12, H - STATE.paddle.h/2 - 12); }

  window.addEventListener('mousemove', e => {
    const rect = canvas.getBoundingClientRect();
    const y = (e.clientY - rect.top) * DPR;
    setTargetFromY(y);
  });
  // Touch
  canvas.addEventListener('touchstart', e => { if (audioCtx.state === 'suspended') audioCtx.resume(); });
  canvas.addEventListener('touchmove', e => {
    const t = e.touches[0]; if (!t) return;
    const rect = canvas.getBoundingClientRect();
    const y = (t.clientY - rect.top) * DPR; setTargetFromY(y);
  }, {passive:true});

  // Keyboard
  const keys = new Set();
  window.addEventListener('keydown', (e)=>{
    if (['ArrowUp','ArrowDown','w','W','s','S','p','P','r','R',' '].includes(e.key)) e.preventDefault();
    keys.add(e.key);
    if (e.key==='p' || e.key==='P') togglePause();
    if (e.key==='r' || e.key==='R') restart();
    if (e.key===' ') { if (!STATE.running && !STATE.countdown) startCountdown(); }
  });
  window.addEventListener('keyup', (e)=> keys.delete(e.key));

  function handleKeyboard(dt){
    const speed = 900; let dir = 0;
    if (keys.has('ArrowUp') || keys.has('w') || keys.has('W')) dir -= 1;
    if (keys.has('ArrowDown') || keys.has('s') || keys.has('S')) dir += 1;
    if (dir !== 0) {
      targetY = clamp(STATE.player.y + dir * speed * dt, STATE.paddle.h/2 + 12, H - STATE.paddle.h/2 - 12);
    }
  }

  function togglePause(){
    STATE.paused = !STATE.paused;
    pauseBtn.textContent = STATE.paused ? 'Resume' : 'Pause';
    center.innerHTML = STATE.paused ? `<h1>PAUSED</h1><p>Press P to resume</p>` : '';
  }
  pauseBtn.onclick = togglePause;
  restartBtn.onclick = restart;

  // ====== AI ======
  function aiUpdate(dt) {
    const b = STATE.ball; const p = STATE.paddle; const ai = STATE.ai;
    const aiX = p.margin + p.w; // left paddle front face

    // Predict where ball will cross AI's x using simple kinematics + wall bounces
    let px = b.x, py = b.y, vx = b.vx, vy = b.vy;
    const targetX = aiX + 6; // a bit inside paddle

    // If ball moving left, predict; else slowly center
    let predictedY = H/2;
    if (vx < 0) {
      while ((vx < 0 && px > targetX) || (vx > 0 && px < targetX)) {
        const t = Math.abs((targetX - px) / vx);
        py += vy * t; px = targetX; // fast-forward to cross
        // reflect off top/bottom virtual walls
        if (py < b.r) { py = b.r + (b.r - py); vy *= -1; }
        if (py > H - b.r) { py = (H - b.r) - (py - (H - b.r)); vy *= -1; }
        break;
      }
      predictedY = py;
    }

    // Difficulty: reaction delay and miss jitter reduce over time / score
    const rounds = STATE.player.score + STATE.ai.score;
    const reaction = clamp(0.12 - rounds * 0.006, 0.028, 0.12); // seconds
    const maxSpeed = 780 + rounds * 20; // AI paddle speed
    const miss = ai.missFactor * (1.0 - Math.min(0.75, rounds*0.04));

    const desired = clamp(predictedY + (Math.random()*2-1)*H*0.06*miss, p.h/2 + 12, H - p.h/2 - 12);
    const diff = desired - ai.y; const step = clamp(diff, -maxSpeed*dt, maxSpeed*dt);
    ai.y += step; ai.vy = step / dt;
  }

  // ====== Update ======
  function update(dt) {
    handleKeyboard(dt);

    // Smoothly move player paddle toward targetY
    const kFollow = 18; // higher = tighter follow
    STATE.player.y += (targetY - STATE.player.y) * (1 - Math.exp(-kFollow*dt));

    // AI update
    aiUpdate(dt);

    // Ball physics
    if (STATE.running) {
      const b = STATE.ball; const p = STATE.paddle;
      b.x += b.vx * dt; b.y += b.vy * dt;

      // Top/bottom walls
      if (b.y < b.r) { b.y = b.r; b.vy *= -1; beep(520,.05,'triangle',.02); }
      else if (b.y > H - b.r) { b.y = H - b.r; b.vy *= -1; beep(520,.05,'triangle',.02); }

      // Paddles
      const playerX = W - p.margin - p.w;
      const aiX = p.margin;

      // Helper for collision with spin
      function collidePaddle(px, py, isPlayer) {
        // Check AABB first
        if (b.x + b.r < px || b.x - b.r > px + p.w) return false;
        if (b.y + b.r < py - p.h/2 || b.y - b.r > py + p.h/2) return false;
        // Move ball outside paddle and reflect
        if (isPlayer) b.x = px - b.r; else b.x = px + p.w + b.r;
        b.vx *= -1;

        // Add spin based on impact position and paddle velocity
        const rel = clamp((b.y - py) / (p.h/2), -1, 1);
        const add = rel * 420 + (isPlayer ? STATE.player.vy : STATE.ai.vy) * 0.18;
        b.vy = clamp(b.vy + add, -STATE.ball.max, STATE.ball.max);

        // Speed up each hit
        const factor = 1.06; const speed = Math.hypot(b.vx, b.vy) * factor;
        const cap = STATE.ball.max;
        const ang = Math.atan2(b.vy, b.vx);
        b.vx = Math.cos(ang) * Math.min(cap, speed);
        b.vy = Math.sin(ang) * Math.min(cap, speed);

        spawnParticles(b.x, b.y, isPlayer? '#22d3ee' : '#94a3b8', 14);
        beep(isPlayer? 880:740, .04, 'square', .03);
        return true;
      }

      // Update paddles' instantaneous vy for spin calc
      STATE.player.vy = (STATE.player.y - (STATE.player._py||STATE.player.y)) / dt; STATE.player._py = STATE.player.y;
      STATE.ai.vy = (STATE.ai.y - (STATE.ai._py||STATE.ai.y)) / dt; STATE.ai._py = STATE.ai.y;

      collidePaddle(playerX, STATE.player.y, true);
      collidePaddle(aiX, STATE.ai.y, false);

      // Goals
      if (b.x < -30) {
        STATE.player.score++; updateScore(); spawnParticles(20,H/2,'#22c55e',28); beep(1200,.12,'triangle',.04);
        if (!winCheck()) { STATE.serving = -1; STATE.running = false; startCountdown(); }
      }
      if (b.x > W + 30) {
        STATE.ai.score++; updateScore(); spawnParticles(W-20,H/2,'#f43f5e',28); beep(300,.14,'sawtooth',.04);
        if (!winCheck()) { STATE.serving = 1; STATE.running = false; startCountdown(); }
      }
    }
  }

  // ====== Render ======
  function render() {
    drawCourt();

    // center line glow when ball near mid
    const b = STATE.ball; if (STATE.running) {
      ctx.globalAlpha = clamp(1 - Math.abs(b.x - W/2)/(W/2), 0, 0.4);
      ctx.fillStyle = 'rgba(34,211,238,.5)';
      ctx.fillRect(W/2 - 2, 0, 4, H);
      ctx.globalAlpha = 1;
    }

    // Paddles & Ball
    drawPaddle(STATE.paddle.margin, STATE.ai.y, false);
    drawPaddle(W - STATE.paddle.margin - STATE.paddle.w, STATE.player.y, true);
    drawBall();

    // Particles
    stepParticles(DELTA);
  }

  // ====== Game Loop ======
  let DELTA = 0; let last = performance.now(); let fpsAccum=0, fpsCount=0;
  function loop(ts){
    if (STATE.paused) { requestAnimationFrame(loop); return; }
    DELTA = Math.min(0.033, (ts - last) / 1000); last = ts;

    update(DELTA);
    render();

    // FPS meter (lightweight)
    fpsAccum += DELTA; fpsCount++;
    if (fpsAccum >= 0.5) { STATE.fps = Math.round(fpsCount / fpsAccum); fpsLabel.textContent = STATE.fps + ' FPS'; fpsAccum = 0; fpsCount = 0; }

    requestAnimationFrame(loop);
  }


  function init(){
    resize();
    STATE.player.y = H/2; STATE.ai.y = H/2; updateScore();
    STATE.running = false; STATE.paused = false; center.innerHTML = '';
    STATE.serving = Math.random() > .5 ? 1 : -1;
    startCountdown();
  }

  init();
  requestAnimationFrame(loop);


  window.addEventListener('pointerdown', ()=>{ if (audioCtx.state === 'suspended') audioCtx.resume(); }, {once:true});
  </script>
</body>
</html>
