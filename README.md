<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no" />
<title>Orbit Shield</title>
<style>
  html, body {
    margin: 0; padding: 0; height: 100%;
    background: radial-gradient(1200px 800px at 50% 40%, #0b132b 0%, #060a18 60%, #04060f 100%);
    color: #e6f1ff; font-family: system-ui, -apple-system, Segoe UI, Roboto, Arial, sans-serif;
    overflow: hidden; touch-action: none;
  }
  #ui {
    position: fixed; top: 0; left: 0; right: 0;
    display: flex; justify-content: space-between; align-items: center;
    padding: 10px 14px; pointer-events: none; user-select: none;
    text-shadow: 0 2px 8px rgba(0,0,0,0.6);
  }
  .pill {
    background: rgba(255,255,255,0.08);
    border: 1px solid rgba(255,255,255,0.12);
    padding: 6px 10px; border-radius: 999px; font-weight: 600; letter-spacing: 0.3px;
    pointer-events: auto;
  }
  #btns {
    position: fixed; bottom: 10px; left: 0; right: 0;
    display: flex; gap: 10px; justify-content: center; pointer-events: none;
  }
  .btn {
    pointer-events: auto; border: 1px solid rgba(255,255,255,0.18);
    background: rgba(255,255,255,0.1); padding: 10px 14px; border-radius: 10px; font-weight: 700;
    backdrop-filter: blur(6px);
  }
  .btn:active { transform: translateY(1px); }
  #centerMsg { position: fixed; inset: 0; display: grid; place-items: center; text-align: center; padding: 24px; pointer-events: none; }
  #centerMsg .card { display: inline-block; background: rgba(0,0,0,0.35); border: 1px solid rgba(255,255,255,0.15); padding: 16px 18px; border-radius: 14px; max-width: 520px; }
  .title { font-size: 28px; margin-bottom: 6px; }
  .subtitle { opacity: 0.85; line-height: 1.4; }
  .mono { font-family: ui-monospace, SFMono-Regular, Menlo, Consolas, monospace; }
  canvas { display: block; width: 100vw; height: 100vh; }
  a { color: #9bdcff; }
</style>
</head>
<body>
<canvas id="game"></canvas>

<div id="ui">
  <div class="pill mono" id="score">Score: 0</div>
  <div class="pill mono" id="mult">x1</div>
  <div class="pill mono" id="health">♥♥♥</div>
</div>

<div id="btns">
  <button class="btn" id="pauseBtn" aria-label="Pause">⏸️ Pause</button>
  <button class="btn" id="restartBtn" aria-label="Restart">🔄 Restart</button>
</div>

<div id="centerMsg">
  <div class="card" id="helpCard">
    <div class="title">🛡️ Orbit Shield</div>
    <div class="subtitle">
      Drag anywhere to rotate the shield around the planet.<br/>
      Block meteors, keep the core safe, and rack up combos.<br/>
      Tip: Quick, precise blocks raise your multiplier.
      <br/><br/>
      גרור עם האגודל כדי לסובב את המגן. חסום מטאורים ושמור על הליבה.
    </div>
    <div style="margin-top:12px; font-size:14px; opacity:0.9">
      Tap to start • הקש כדי להתחיל
    </div>
  </div>
</div>

<script>
(() => {
  const canvas = document.getElementById('game');
  const ctx = canvas.getContext('2d');
  let W, H, DPR = Math.max(1, Math.min(2, window.devicePixelRatio||1));
  function resize() {
    W = Math.floor(window.innerWidth);
    H = Math.floor(window.innerHeight);
    canvas.width = Math.floor(W * DPR);
    canvas.height = Math.floor(H * DPR);
    ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
  }
  window.addEventListener('resize', resize, {passive:true});
  resize();

  // UI
  const scoreEl = document.getElementById('score');
  const multEl  = document.getElementById('mult');
  const healthEl= document.getElementById('health');
  const pauseBtn= document.getElementById('pauseBtn');
  const restartBtn=document.getElementById('restartBtn');
  const centerMsg= document.getElementById('centerMsg');
  const helpCard = document.getElementById('helpCard');

  // Game state
  let running = false;
  let paused = false;
  let time = 0;
  let lastTs = 0;
  let rng = Math.random;

  let score = 0;
  let health = 3;
  let combo = 0;
  let multiplier = 1;

  const core = {x:0, y:0, r: 30};
  const shield = {angle: 0, width: Math.PI/6, radius: 90};
  const meteors = [];
  let spawnTimer = 0;

  function resetGame() {
    score = 0;
    health = 3;
    combo = 0;
    multiplier = 1;
    shield.angle = -Math.PI/2;
    meteors.length = 0;
    spawnTimer = 0;
    time = 0;
    lastTs = 0;
    updateUI();
    paused = false;
  }

  function updateUI() {
    scoreEl.textContent = 'Score: ' + Math.floor(score);
    multEl.textContent  = 'x' + multiplier;
    healthEl.textContent= '♥'.repeat(health) + ' '.repeat(Math.max(0,3-health));
  }

  // Input: drag to rotate
  let dragging = false;
  function angleTo(x, y) { return Math.atan2(y - core.y, x - core.x); }
  function onPointerDown(e) {
    const p = getPoint(e);
    angleTo(p.x, p.y);
    dragging = true;
    if (!running) {
      centerMsg.style.display = 'none';
      running = true;
    }
  }
  function onPointerMove(e) {
    if (!dragging) return;
    const p = getPoint(e);
    const a = angleTo(p.x, p.y);
    shield.angle = a;
  }
  function onPointerUp() { dragging = false; }
  function getPoint(e) {
    const t = (e.touches && e.touches[0]) || e.changedTouches?.[0] || e;
    const rect = canvas.getBoundingClientRect();
    return { x: t.clientX - rect.left, y: t.clientY - rect.top };
  }
  canvas.addEventListener('mousedown', onPointerDown);
  canvas.addEventListener('mousemove', onPointerMove);
  window.addEventListener('mouseup', onPointerUp);
  canvas.addEventListener('touchstart', onPointerDown, {passive:true});
  canvas.addEventListener('touchmove', onPointerMove, {passive:true});
  window.addEventListener('touchend', onPointerUp, {passive:true});

  pauseBtn.addEventListener('click', () => {
    if (!running) return;
    paused = !paused;
    pauseBtn.textContent = paused ? '▶️ Resume' : '⏸️ Pause';
  });
  restartBtn.addEventListener('click', () => {
    resetGame();
    centerMsg.style.display = 'none';
    running = true;
  });

  function spawnMeteor() {
    const ang = rng() * Math.PI * 2;
    const dist = Math.max(W,H) * 0.6 + 120;
    const speed = 90 + rng() * 80 + (score * 0.01); // scales with score
    const size = 10 + rng()*14;
    const x = core.x + Math.cos(ang) * dist;
    const y = core.y + Math.sin(ang) * dist;
    const dir = Math.atan2(core.y - y, core.x - x);
    meteors.push({ x, y, r: size, speed, dir, exploded: false, v: {x: Math.cos(dir)*speed, y: Math.sin(dir)*speed} });
  }

  function update(dt) {
    time += dt;
    spawnTimer -= dt;
    const spawnRate = Math.max(0.2, 1.2 - Math.min(1, score/400)); // faster spawns as score grows
    if (spawnTimer <= 0) {
      spawnMeteor();
      if (score > 120) spawnMeteor();
      spawnTimer = spawnRate;
    }

    for (let i = meteors.length-1; i >= 0; i--) {
      const m = meteors[i];
      m.x += m.v.x * dt;
      m.y += m.v.y * dt;

      const dx = m.x - core.x, dy = m.y - core.y;
      const dist = Math.hypot(dx, dy);
      const theta = Math.atan2(dy, dx);

      const ringMin = shield.radius - 12;
      const ringMax = shield.radius + 12;

      let dAng = theta - shield.angle;
      dAng = Math.atan2(Math.sin(dAng), Math.cos(dAng));
      const withinArc = Math.abs(dAng) < shield.width * 0.6;

      if (!m.exploded && dist < ringMax && dist > ringMin && withinArc) {
        m.exploded = true;
        score += 10 * multiplier;
        combo += 1;
        if (combo % 4 === 0) multiplier = Math.min(10, multiplier + 1);
        const outAng = theta + (Math.random()-0.5)*0.6;
        m.v.x = Math.cos(outAng) * (m.speed * 1.2);
        m.v.y = Math.sin(outAng) * (m.speed * 1.2);
        m.r *= 0.75;
        flash(0.18);
        setTimeout(() => {
          const idx = meteors.indexOf(m);
          if (idx >= 0) meteors.splice(idx,1);
        }, 400);
      }

      if (!m.exploded && dist < core.r + m.r) {
        meteors.splice(i,1);
        combo = 0; multiplier = 1;
        health -= 1;
        shake(10, 280);
        flash(0.35, 'rgba(255,60,60,0.3)');
        if (health <= 0) { gameOver(); return; }
      }

      if (m.x < -200 || m.x > W+200 || m.y < -200 || m.y > H+200) {
        meteors.splice(i,1);
      }
    }

    updateUI();
  }

  // FX
  let shakeTime = 0, shakeMag = 0;
  function shake(mag, ms) { shakeMag = mag; shakeTime = ms/1000; }
  let flashAlpha = 0, flashCol = 'white';
  function flash(a, col='white') { flashAlpha = a; flashCol = col; }

  function draw() {
    if (shakeTime > 0) {
      shakeTime -= dtGlobal;
      const s = shakeMag * (shakeTime);
      ctx.setTransform(DPR, 0, 0, DPR, (Math.random()*s - s/2), (Math.random()*s - s/2));
    } else {
      ctx.setTransform(DPR, 0, 0, DPR, 0, 0);
    }

    ctx.clearRect(0,0,W,H);
    drawStars();

    core.x = W/2; core.y = H/2;
    const grd = ctx.createRadialGradient(core.x, core.y, 2, core.x, core.y, core.r);
    grd.addColorStop(0, '#94c5ff'); grd.addColorStop(1, '#2b6cb0');
    ctx.fillStyle = grd;
    ctx.beginPath(); ctx.arc(core.x, core.y, core.r, 0, Math.PI*2); ctx.fill();

    ctx.save();
    ctx.translate(core.x, core.y);
    ctx.rotate(shield.angle);
    ctx.strokeStyle = 'rgba(180,220,255,0.22)';
    ctx.lineWidth = 10;
    ctx.beginPath(); ctx.arc(0,0, shield.radius, 0, Math.PI*2); ctx.stroke();
    ctx.strokeStyle = 'rgba(150,220,255,0.9)';
    ctx.lineWidth = 14;
    ctx.beginPath(); ctx.arc(0,0, shield.radius, -shield.width/2, shield.width/2); ctx.stroke();
    ctx.restore();

    for (const m of meteors) {
      const g = ctx.createRadialGradient(m.x, m.y, 2, m.x, m.y, m.r);
      g.addColorStop(0, '#ffd29b'); g.addColorStop(1, '#b85c2b');
      ctx.fillStyle = g;
      ctx.beginPath(); ctx.arc(m.x, m.y, m.r, 0, Math.PI*2); ctx.fill();

      ctx.strokeStyle = 'rgba(255,200,120,0.35)';
      ctx.lineWidth = 2;
      ctx.beginPath();
      const tx = m.x - m.v.x * 0.03;
      const ty = m.y - m.v.y * 0.03;
      ctx.moveTo(m.x, m.y); ctx.lineTo(tx, ty); ctx.stroke();
    }

    if (flashAlpha > 0) {
      ctx.fillStyle = flashCol;
      ctx.globalAlpha = flashAlpha;
      ctx.fillRect(0,0,W,H);
      ctx.globalAlpha = 1;
      flashAlpha *= 0.9;
      if (flashAlpha < 0.02) flashAlpha = 0;
    }
  }

  // starfield
  const stars = Array.from({length: 200}, () => ({ x: Math.random(), y: Math.random(), s: Math.random() }));
  function drawStars() {
    ctx.save();
    for (const st of stars) {
      const x = st.x * W, y = st.y * H, r = st.s * 1.4 + 0.2;
      ctx.fillStyle = 'rgba(255,255,255,' + (0.2 + st.s*0.8) + ')';
      ctx.beginPath(); ctx.arc(x, y, r, 0, Math.PI*2); ctx.fill();
    }
    ctx.restore();
  }

  function gameOver() {
    running = false; paused = false; updateUI();
    centerMsg.style.display = 'grid';
    helpCard.innerHTML = `
      <div class="title">Game Over</div>
      <div class="subtitle">Score: <b>${Math.floor(score)}</b> • Multiplier Max: x${multiplier}</div>
      <div style="margin-top:10px; opacity:0.9">
        Tap <b>Restart</b> to try again.
      </div>
    `;
  }

  // Main loop
  let dtGlobal = 0;
  function loop(ts) {
    requestAnimationFrame(loop);
    if (!running || paused) { lastTs = ts; return; }
    const dt = Math.min(0.033, (ts - lastTs) / 1000 || 0);
    dtGlobal = dt; lastTs = ts;
    update(dt); draw();
  }
  requestAnimationFrame(loop);

  resetGame();
  centerMsg.style.display = 'grid';

  // iOS: reduce double-tap zoom quirks
  let lastTouchEnd = 0;
  document.addEventListener('touchend', function (e) {
    const now = Date.now();
    if (now - lastTouchEnd <= 300) e.preventDefault();
    lastTouchEnd = now;
  }, {passive:false});
})();
</script>
</body>
</html>
