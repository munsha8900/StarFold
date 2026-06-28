# STARFOLD — Android APK build (GitHub Actions)

A native Android WebView wrapper around the STARFOLD game. Push to GitHub and the included workflow builds a debug APK for you — no Android Studio needed.

## Repo structure

```
starfold-android/
├─ .github/workflows/build.yml
├─ app/
│  ├─ build.gradle
│  └─ src/main/
│     ├─ AndroidManifest.xml
│     ├─ assets/index.html
│     ├─ java/com/markfold/starfold/MainActivity.java
│     └─ res/
│        ├─ drawable/ic_launcher.xml
│        └─ values/themes.xml
├─ build.gradle
├─ settings.gradle
└─ gradle.properties
```

## How to build

1. Create a new GitHub repo (e.g. `starfold-android`).
2. Add every file below at its exact path.
3. Commit and push to the `main` branch.
4. Open the **Actions** tab → wait for "Build STARFOLD APK" to finish (~3–5 min).
5. Open the finished run → download the **starfold-debug-apk** artifact → unzip → `app-debug.apk`.
6. Transfer to your phone and install (enable "Install unknown apps" for your file manager/browser).

> This produces a **debug-signed** APK — perfect for sideloading and testing. For a Play Store release you'd add a release signing config; tell me if you want that set up.

---

## `app/src/main/assets/index.html`

This is the full game, with high-score/mute persistence switched to `localStorage` (which works in a real Android WebView).

```html
<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no, viewport-fit=cover">
<title>STARFOLD</title>
<style>
  @import url('https://fonts.googleapis.com/css2?family=Hanken+Grotesk:wght@400;500;700;800&family=Bricolage+Grotesque:wght@700;800&display=swap');

  * { box-sizing: border-box; -webkit-tap-highlight-color: transparent; }
  html, body {
    margin: 0; padding: 0; height: 100%; width: 100%;
    background: #06080d; overflow: hidden;
    touch-action: none; user-select: none; -webkit-user-select: none;
    -webkit-touch-callout: none;
    font-family: 'Hanken Grotesk', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  }
  #wrap { position: fixed; inset: 0; }
  canvas { display: block; width: 100%; height: 100%; }

  .overlay {
    position: absolute; inset: 0; display: flex; flex-direction: column;
    align-items: center; justify-content: center; gap: 18px;
    text-align: center; pointer-events: none; padding: 24px;
    opacity: 0; transition: opacity .28s ease; visibility: hidden;
  }
  .overlay.show { opacity: 1; visibility: visible; }

  .panel {
    background: rgba(12, 18, 26, 0.55);
    border: 1px solid rgba(25, 227, 200, 0.35);
    border-radius: 22px; padding: 30px 30px 26px;
    backdrop-filter: blur(8px);
    box-shadow: 0 0 40px rgba(25, 227, 200, 0.18), inset 0 0 24px rgba(25,227,200,0.05);
    display: flex; flex-direction: column; align-items: center; gap: 14px;
    max-width: 360px; width: 88%;
  }

  .title {
    font-family: 'Bricolage Grotesque', 'Hanken Grotesk', sans-serif;
    font-weight: 800; font-size: clamp(40px, 12vw, 60px);
    letter-spacing: 2px; margin: 0; line-height: 0.95;
    color: #F4EEE0;
    text-shadow: 0 0 18px rgba(25,227,200,0.6), 0 0 38px rgba(25,227,200,0.35);
  }
  .tagline { font-size: 13px; color: #8fd9cf; letter-spacing: 1.5px; text-transform: uppercase; margin: -4px 0 4px; }

  .cta {
    font-weight: 800; font-size: 16px; letter-spacing: 1.5px;
    color: #06251f; text-transform: uppercase;
    background: linear-gradient(180deg, #5fffe4, #16e0c6);
    padding: 13px 26px; border-radius: 999px;
    box-shadow: 0 0 26px rgba(25,227,200,0.55);
    animation: pulse 1.5s ease-in-out infinite;
  }
  @keyframes pulse { 0%,100%{ transform: scale(1); opacity: .95; } 50%{ transform: scale(1.05); opacity: 1; } }

  .scoreRow { display: flex; gap: 26px; align-items: flex-end; }
  .scoreBox { display: flex; flex-direction: column; align-items: center; gap: 2px; }
  .scoreBox .lbl { font-size: 11px; letter-spacing: 2px; color: #7fb8b0; text-transform: uppercase; }
  .scoreBox .val {
    font-family: 'Bricolage Grotesque', sans-serif; font-weight: 800;
    font-size: 38px; color: #FFD66B; line-height: 1;
    text-shadow: 0 0 16px rgba(255,214,107,0.5);
  }
  .scoreBox.best .val { color: #F4EEE0; text-shadow: 0 0 14px rgba(244,238,224,0.4); }

  .logo { display: flex; align-items: center; gap: 12px; filter: drop-shadow(0 0 14px rgba(250,201,61,0.5)); }
  .logo .mfslot svg { height: 46px; width: auto; display: block; }
  .logo .wm { display: flex; flex-direction: column; align-items: flex-start; line-height: 1; }
  .logo .wm .mk {
    font-family: 'Bricolage Grotesque', sans-serif; font-weight: 800;
    font-size: 19px; letter-spacing: 1px; color: #F4EEE0;
  }
  .logo .wm .dg { font-size: 9px; letter-spacing: 4px; color: #FFD66B; margin-top: 2px; font-weight: 700; }
  .credit { font-size: 11px; color: #6f9a93; letter-spacing: 0.5px; margin-top: 2px; }

  #corner {
    position: absolute; left: 14px; bottom: 14px; display: flex; align-items: center; gap: 8px;
    opacity: 0.55; pointer-events: none; filter: drop-shadow(0 0 9px rgba(250,201,61,0.45));
    transition: opacity .3s ease;
  }
  #corner .mfslot svg { height: 24px; width: auto; display: block; }
  #corner .ct { font-family: 'Bricolage Grotesque', sans-serif; font-weight: 800; font-size: 12px; letter-spacing: 0.5px; color: #d8efe9; }
  #corner.hide { opacity: 0; }

  #mute {
    position: absolute; top: 16px; right: 16px; width: 42px; height: 42px;
    display: flex; align-items: center; justify-content: center;
    border-radius: 50%; background: rgba(12,18,26,0.6);
    border: 1px solid rgba(25,227,200,0.3); cursor: pointer; pointer-events: auto;
    color: #aef0e6; z-index: 5;
  }
  #mute svg { width: 20px; height: 20px; }
</style>
</head>
<body>
<div id="wrap">
  <canvas id="game"></canvas>

  <div id="corner">
    <span class="mfslot"></span>
    <span class="ct">MarkFold</span>
  </div>

  <div id="mute" role="button" aria-label="Toggle sound"></div>

  <div id="startScreen" class="overlay">
    <div class="panel">
      <div class="logo">
        <span class="mfslot"></span>
        <div class="wm"><span class="mk">MARKFOLD</span><span class="dg">DIGITAL</span></div>
      </div>
      <h1 class="title">STARFOLD</h1>
      <div class="tagline">Fold through deep space</div>
      <div class="scoreRow">
        <div class="scoreBox best"><span class="lbl">Best</span><span class="val" id="startBest">0</span></div>
      </div>
      <div class="cta">Tap to Start</div>
      <div class="credit">Created by MarkFold Digital</div>
    </div>
  </div>

  <div id="overScreen" class="overlay">
    <div class="panel">
      <div class="title" style="font-size:clamp(30px,9vw,44px)">GAME OVER</div>
      <div class="scoreRow">
        <div class="scoreBox"><span class="lbl">Score</span><span class="val" id="finalScore">0</span></div>
        <div class="scoreBox best"><span class="lbl">Best</span><span class="val" id="finalBest">0</span></div>
      </div>
      <div class="cta">Tap to Play Again</div>
      <div class="logo" style="margin-top:4px">
        <span class="mfslot"></span>
        <div class="wm"><span class="mk">MARKFOLD</span><span class="dg">DIGITAL</span></div>
      </div>
      <div class="credit">Created by MarkFold Digital</div>
    </div>
  </div>
</div>

<script>
(function () {
  const cv = document.getElementById('game');
  const ctx = cv.getContext('2d');
  const startScreen = document.getElementById('startScreen');
  const overScreen  = document.getElementById('overScreen');
  const corner      = document.getElementById('corner');
  const muteBtn     = document.getElementById('mute');

  const MF_SVG = '<svg viewBox="0 0 604 512" xmlns="http://www.w3.org/2000/svg"><g transform="translate(0,512) scale(0.1,-0.1)" fill="#FAC93D"><path d="M547 5106 c-3 -8 -30 -22 -59 -31 -124 -41 -231 -117 -326 -230 -48 -58 -103 -161 -117 -217 -4 -14 -15 -34 -26 -44 -19 -17 -19 -61 -19 -2001 0 -1961 0 -1983 20 -2003 11 -11 22 -32 26 -47 14 -60 81 -160 154 -233 66 -66 159 -140 177 -140 3 0 27 -13 52 -29 34 -22 72 -34 146 -47 55 -9 120 -20 144 -24 27 -4 69 -3 105 5 34 7 84 15 112 19 33 4 95 28 170 65 98 49 125 67 152 104 19 24 45 51 58 58 35 18 58 60 94 169 l31 95 -1 975 c0 634 3 987 10 1010 6 19 17 72 25 118 8 46 21 95 28 110 117 222 114 217 227 330 54 54 101 91 137 109 31 14 72 40 92 58 20 18 68 49 106 70 39 20 88 52 110 70 22 17 72 49 110 69 39 20 97 58 130 84 74 58 104 77 136 87 13 4 49 31 80 59 31 28 68 59 83 69 33 23 48 58 48 111 -1 53 -27 79 -177 180 -166 111 -213 141 -255 166 -19 11 -46 28 -60 39 -23 17 -180 119 -310 203 -133 85 -277 179 -299 195 -14 10 -54 36 -90 58 -91 56 -172 107 -302 193 -105 68 -197 115 -281 141 -20 7 -42 19 -48 27 -18 20 -385 21 -393 0z"/><path d="M5560 5004 c-42 -9 -129 -43 -158 -63 -15 -10 -70 -44 -122 -76 -52 -32 -112 -69 -132 -81 -21 -13 -57 -36 -80 -51 -24 -14 -59 -36 -78 -48 -19 -12 -69 -44 -110 -71 -41 -27 -86 -55 -100 -62 -14 -7 -32 -20 -41 -28 -8 -8 -19 -14 -23 -14 -4 0 -26 -13 -49 -28 -23 -16 -60 -40 -82 -54 -22 -13 -58 -36 -80 -50 -22 -13 -58 -36 -80 -50 -22 -13 -58 -36 -80 -50 -22 -14 -75 -47 -119 -74 -43 -27 -113 -72 -156 -101 -43 -28 -93 -60 -110 -70 -52 -29 -65 -37 -111 -68 -24 -15 -60 -38 -79 -50 -19 -12 -69 -44 -110 -71 -41 -27 -79 -51 -85 -53 -5 -2 -46 -27 -90 -56 -112 -72 -136 -87 -165 -105 -53 -31 -156 -97 -232 -147 -43 -29 -81 -53 -84 -53 -2 0 -17 -8 -32 -19 -48 -33 -230 -149 -322 -206 -70 -43 -126 -79 -167 -107 -23 -15 -45 -28 -51 -28 -5 0 -17 -9 -27 -20 -10 -11 -27 -23 -39 -26 -11 -4 -34 -16 -51 -28 -16 -12 -37 -26 -46 -31 -107 -67 -241 -193 -278 -261 -12 -21 -25 -46 -31 -54 -5 -8 -21 -42 -35 -75 -52 -117 -55 -169 -55 -940 0 -766 -1 -747 54 -773 24 -12 34 -12 70 2 23 9 60 33 82 53 21 20 68 51 104 69 36 18 88 51 115 72 28 22 100 68 160 103 61 34 124 76 142 94 17 18 51 42 75 54 24 11 68 39 98 60 30 22 82 56 116 76 33 19 69 44 79 55 9 10 39 26 66 35 26 9 63 32 82 51 36 37 191 134 214 134 7 0 28 15 46 33 19 18 67 53 108 78 41 25 88 54 105 65 55 34 218 142 259 172 22 17 60 40 85 52 25 12 74 44 110 70 36 25 88 59 115 75 28 15 61 38 75 50 29 27 117 75 136 75 8 0 22 8 32 17 9 10 41 33 70 52 29 18 60 44 70 56 10 13 49 37 86 54 37 17 77 40 89 51 49 45 161 120 192 129 19 5 60 31 93 57 68 56 139 103 296 194 62 36 139 86 171 111 33 25 87 61 120 78 33 18 85 56 115 85 30 29 76 69 103 90 26 20 47 43 47 50 0 7 22 42 48 78 36 47 51 77 55 112 4 25 13 59 22 75 12 25 15 99 15 473 0 475 -5 555 -40 633 -32 71 -133 166 -207 194 -59 23 -137 33 -183 25z"/><path d="M5505 2575 c-39 -19 -108 -60 -155 -90 -47 -30 -101 -64 -120 -75 -19 -11 -46 -28 -60 -38 -14 -11 -43 -30 -65 -44 -22 -14 -94 -60 -160 -103 -66 -43 -136 -88 -155 -100 -19 -12 -89 -57 -155 -100 -147 -96 -277 -180 -327 -212 -96 -59 -115 -71 -142 -89 -16 -10 -36 -24 -45 -29 -9 -6 -50 -32 -91 -59 -41 -27 -91 -59 -110 -71 -162 -102 -301 -198 -319 -221 -27 -34 -27 -84 0 -118 22 -28 41 -42 159 -118 36 -23 117 -75 180 -116 111 -72 147 -95 228 -145 20 -13 50 -32 66 -43 16 -10 37 -24 46 -29 10 -6 35 -22 56 -36 22 -14 74 -49 117 -77 44 -29 82 -52 87 -52 4 0 13 -7 21 -16 7 -8 29 -24 48 -35 20 -11 54 -32 77 -47 83 -55 150 -97 176 -112 15 -8 58 -32 94 -53 37 -21 89 -42 116 -47 28 -6 73 -17 101 -25 58 -17 134 -19 192 -5 22 6 69 14 104 19 40 6 71 17 85 30 12 12 43 28 68 37 29 9 67 36 102 70 31 30 74 65 96 79 22 14 40 31 40 38 0 15 46 104 71 136 10 12 21 39 25 60 3 20 13 60 21 89 12 44 14 160 11 770 l-4 717 -26 59 c-38 89 -121 175 -201 209 -96 39 -162 37 -252 -8z"/></g></svg>';
  document.querySelectorAll('.mfslot').forEach(function (el) { el.innerHTML = MF_SVG; });

  const TEAL = '#16e0c6', TEAL_HI = '#5fffe4', BUTTER = '#FFD66B', BONE = '#F4EEE0', SKY = '#7fd4f0';

  let W = 0, H = 0, dpr = 1;
  let state = 'ready';
  let best = 0, score = 0;
  let muted = false;
  let deadAt = 0;

  let bx = 0, by = 0, vy = 0;
  let playH = 0, groundH = 0, R = 14;
  let G = 0, IMP = 0, MAXFALL = 0;
  let pipeW = 0, spacing = 0, gapBase = 0, gapMin = 0, spd0 = 0;
  let pipes = [];
  let trail = [];
  let stars = [];
  let shake = 0, flash = 0;
  let t = 0;

  function resize() {
    const r = cv.getBoundingClientRect();
    W = r.width; H = r.height;
    dpr = Math.min(window.devicePixelRatio || 1, 2);
    cv.width = Math.round(W * dpr); cv.height = Math.round(H * dpr);
    ctx.setTransform(dpr, 0, 0, dpr, 0, 0);

    groundH = H * 0.07;
    playH = H - groundH;
    bx = W * 0.28;
    R = clamp(Math.min(W, H) * 0.028, 10, 20);
    G = H * 3.0;
    IMP = -H * 0.66;
    MAXFALL = H * 0.95;
    pipeW = clamp(W * 0.18, 50, 95);
    spacing = pipeW + clamp(W * 0.5, 170, 320);
    gapBase = H * 0.30;
    gapMin = H * 0.20;
    spd0 = Math.max(W * 0.5, H * 0.30);

    if (state === 'ready') by = playH * 0.5;
    buildStars();
  }
  function clamp(v, a, b) { return v < a ? a : v > b ? b : v; }

  function buildStars() {
    stars = [];
    const n = Math.round((W * H) / 9000);
    for (let i = 0; i < n; i++) {
      const layer = Math.random() < 0.6 ? 0 : 1;
      stars.push({
        x: Math.random() * W, y: Math.random() * H,
        r: layer ? Math.random() * 1.6 + 0.8 : Math.random() * 1.0 + 0.3,
        f: layer ? 0.55 : 0.22,
        c: Math.random() < 0.8 ? BONE : SKY,
        tw: Math.random() * 6.28
      });
    }
  }

  function level() { return Math.floor(score / 4); }
  function curSpeed() { return Math.min(spd0 * (1 + level() * 0.05), spd0 * 1.7); }
  function curGap() { return Math.max(gapMin, gapBase - level() * (H * 0.014)); }

  function newGame() {
    score = 0; vy = 0; by = playH * 0.5;
    pipes = []; trail = []; shake = 0; flash = 0;
    const g = curGap();
    let firstX = W * 1.05;
    for (let i = 0; i < 3; i++) pipes.push(makePipe(firstX + i * spacing, g));
    state = 'playing';
    flap();
    syncScreens();
  }
  function makePipe(x, g) {
    const gf = g / playH;
    const lo = gf / 2 + 0.08, hi = 1 - gf / 2 - 0.08;
    const cf = lo + Math.random() * (hi - lo);
    return { x, cf, scored: false };
  }
  function flap() { vy = IMP; if (state === 'playing') beep('flap'); }

  function die() {
    state = 'dead';
    deadAt = t;
    shake = 1; flash = 1;
    beep('crash');
    if (score > best) { best = score; saveBest(); }
    syncScreens();
  }

  function syncScreens() {
    startScreen.classList.toggle('show', state === 'ready');
    overScreen.classList.toggle('show', state === 'dead');
    corner.classList.toggle('hide', state === 'playing');
    document.getElementById('startBest').textContent = best;
    document.getElementById('finalScore').textContent = score;
    document.getElementById('finalBest').textContent = best;
  }

  function update(dt) {
    t += dt;
    const sp = curSpeed();
    for (const s of stars) {
      s.x -= sp * s.f * dt * (state === 'playing' ? 1 : 0.4);
      if (s.x < -2) { s.x = W + 2; s.y = Math.random() * H; }
    }
    if (state !== 'playing') return;

    vy += G * dt;
    if (vy > MAXFALL) vy = MAXFALL;
    by += vy * dt;
    if (by < R) { by = R; vy = 0; }

    for (const p of pipes) p.x -= sp * dt;
    while (pipes.length && pipes[0].x < -pipeW) pipes.shift();
    const last = pipes[pipes.length - 1];
    if (!last || last.x <= W - spacing) {
      pipes.push(makePipe((last ? last.x : W) + spacing, curGap()));
    }

    const g = curGap();
    for (const p of pipes) {
      const cy = clamp(p.cf * playH, g / 2 + 4, playH - g / 2 - 4);
      const topH = cy - g / 2;
      const botY = cy + g / 2;
      if (circRect(bx, by, R, p.x, 0, pipeW, topH) ||
          circRect(bx, by, R, p.x, botY, pipeW, playH - botY)) {
        return die();
      }
      if (!p.scored && p.x + pipeW < bx) { p.scored = true; score++; beep('score'); }
    }
    if (by + R >= playH) { by = playH - R; return die(); }

    trail.push({ x: bx, y: by });
    if (trail.length > 16) trail.shift();
  }

  function circRect(cx, cy, r, rx, ry, rw, rh) {
    const nx = clamp(cx, rx, rx + rw), ny = clamp(cy, ry, ry + rh);
    const dx = cx - nx, dy = cy - ny;
    return dx * dx + dy * dy < r * r;
  }

  function render() {
    ctx.save();
    if (shake > 0) {
      const m = shake * 9;
      ctx.translate((Math.random() - 0.5) * m, (Math.random() - 0.5) * m);
      shake = Math.max(0, shake - 0.06);
    }
    const bg = ctx.createLinearGradient(0, 0, 0, H);
    bg.addColorStop(0, '#0a1018'); bg.addColorStop(0.6, '#080d15'); bg.addColorStop(1, '#05080e');
    ctx.fillStyle = bg; ctx.fillRect(-20, -20, W + 40, H + 40);

    for (const s of stars) {
      const a = (s.f) * (0.6 + 0.4 * Math.sin(t * 2 + s.tw));
      ctx.globalAlpha = a; ctx.fillStyle = s.c;
      ctx.beginPath(); ctx.arc(s.x, s.y, s.r, 0, 6.2832); ctx.fill();
    }
    ctx.globalAlpha = 1;

    const g = curGap();
    for (const p of pipes) {
      const cy = clamp(p.cf * playH, g / 2 + 4, playH - g / 2 - 4);
      const topH = cy - g / 2, botY = cy + g / 2;
      neonPipe(p.x, 0, pipeW, topH, true);
      neonPipe(p.x, botY, pipeW, playH - botY, false);
    }

    ctx.save();
    ctx.shadowColor = TEAL; ctx.shadowBlur = 16;
    ctx.strokeStyle = 'rgba(25,227,200,0.7)'; ctx.lineWidth = 2;
    ctx.beginPath(); ctx.moveTo(0, playH); ctx.lineTo(W, playH); ctx.stroke();
    ctx.restore();
    ctx.fillStyle = 'rgba(8,14,20,0.85)'; ctx.fillRect(0, playH, W, groundH + 2);

    for (let i = 0; i < trail.length; i++) {
      const tp = trail[i], a = (i / trail.length) * 0.5;
      ctx.globalAlpha = a; ctx.fillStyle = TEAL;
      ctx.beginPath(); ctx.arc(tp.x, tp.y, R * (0.3 + 0.6 * i / trail.length), 0, 6.2832); ctx.fill();
    }
    ctx.globalAlpha = 1;

    let drawY = by;
    if (state !== 'playing') drawY = playH * 0.5 + Math.sin(t * 2.2) * (H * 0.018);
    drawBird(bx, drawY);

    if (state === 'playing') {
      ctx.save();
      ctx.textAlign = 'center';
      ctx.font = '800 ' + Math.round(clamp(H * 0.08, 34, 64)) + "px 'Bricolage Grotesque', sans-serif";
      ctx.fillStyle = BONE; ctx.shadowColor = BUTTER; ctx.shadowBlur = 18;
      ctx.fillText(score, W / 2, H * 0.16);
      ctx.restore();
    }

    if (flash > 0) {
      ctx.fillStyle = 'rgba(255,90,80,' + (flash * 0.4) + ')';
      ctx.fillRect(-20, -20, W + 40, H + 40);
      flash = Math.max(0, flash - 0.05);
    }
    ctx.restore();
  }

  function neonPipe(x, y, w, h, top) {
    if (h <= 0) return;
    ctx.save();
    const grad = ctx.createLinearGradient(x, 0, x + w, 0);
    grad.addColorStop(0, '#241a06'); grad.addColorStop(0.5, '#5a4310'); grad.addColorStop(1, '#241a06');
    ctx.fillStyle = grad;
    roundRect(x, y, w, h, 9); ctx.fill();
    ctx.shadowColor = BUTTER; ctx.shadowBlur = 14;
    ctx.lineWidth = 2.5; ctx.strokeStyle = '#FFCB52';
    roundRect(x, y, w, h, 9); ctx.stroke();
    ctx.shadowBlur = 18;
    const lipY = top ? y + h - 6 : y;
    ctx.fillStyle = '#FFE39A';
    roundRect(x + 2, lipY, w - 4, 6, 3); ctx.fill();
    ctx.restore();
  }

  function drawBird(x, y) {
    ctx.save();
    ctx.shadowColor = TEAL; ctx.shadowBlur = R * 2.0;
    const grad = ctx.createRadialGradient(x - R * 0.3, y - R * 0.3, R * 0.15, x, y, R);
    grad.addColorStop(0, '#E6FFFA'); grad.addColorStop(0.45, TEAL_HI); grad.addColorStop(1, '#11c9b0');
    ctx.fillStyle = grad;
    ctx.beginPath(); ctx.arc(x, y, R, 0, 6.2832); ctx.fill();
    ctx.shadowBlur = 0; ctx.fillStyle = 'rgba(255,255,255,0.85)';
    ctx.beginPath(); ctx.arc(x - R * 0.25, y - R * 0.25, R * 0.28, 0, 6.2832); ctx.fill();
    ctx.restore();
  }

  function roundRect(x, y, w, h, r) {
    r = Math.min(r, w / 2, h / 2);
    ctx.beginPath();
    if (ctx.roundRect) { ctx.roundRect(x, y, w, h, r); return; }
    ctx.moveTo(x + r, y);
    ctx.arcTo(x + w, y, x + w, y + h, r);
    ctx.arcTo(x + w, y + h, x, y + h, r);
    ctx.arcTo(x, y + h, x, y, r);
    ctx.arcTo(x, y, x + w, y, r);
    ctx.closePath();
  }

  let lastT = performance.now(), acc = 0;
  const STEP = 1 / 120;
  function frame(now) {
    let dt = (now - lastT) / 1000; lastT = now;
    if (dt > 0.25) dt = 0.25;
    acc += dt;
    let guard = 0;
    while (acc >= STEP && guard < 8) { update(STEP); acc -= STEP; guard++; }
    if (guard >= 8) acc = 0;
    render();
    requestAnimationFrame(frame);
  }

  let actx = null;
  function initAudio() {
    if (!actx) { try { actx = new (window.AudioContext || window.webkitAudioContext)(); } catch (e) {} }
    if (actx && actx.state === 'suspended') actx.resume();
  }
  function beep(kind) {
    if (muted || !actx) return;
    const now = actx.currentTime;
    if (kind === 'flap') tone(620, 0.09, 'triangle', 0.18, 980);
    else if (kind === 'score') { tone(740, 0.08, 'sine', 0.2); setTimeout(() => tone(1050, 0.1, 'sine', 0.2), 60); }
    else if (kind === 'crash') {
      tone(220, 0.25, 'sawtooth', 0.25, 60);
      try {
        const b = actx.createBuffer(1, actx.sampleRate * 0.2, actx.sampleRate);
        const d = b.getChannelData(0);
        for (let i = 0; i < d.length; i++) d[i] = (Math.random() * 2 - 1) * (1 - i / d.length);
        const src = actx.createBufferSource(); src.buffer = b;
        const ng = actx.createGain(); ng.gain.value = 0.18;
        src.connect(ng); ng.connect(actx.destination); src.start(now);
      } catch (e) {}
    }
  }
  function tone(freq, dur, type, vol, slideTo) {
    if (!actx) return;
    const now = actx.currentTime;
    const o = actx.createOscillator(), gn = actx.createGain();
    o.type = type; o.frequency.setValueAtTime(freq, now);
    if (slideTo) o.frequency.exponentialRampToValueAtTime(slideTo, now + dur);
    gn.gain.setValueAtTime(vol, now);
    gn.gain.exponentialRampToValueAtTime(0.0001, now + dur);
    o.connect(gn); gn.connect(actx.destination);
    o.start(now); o.stop(now + dur + 0.02);
  }

  // ---------- storage (localStorage — works in a real Android WebView) ----------
  function loadSaved() {
    try { best = parseInt(localStorage.getItem('starfold_best')) || 0; } catch (e) {}
    try { muted = localStorage.getItem('starfold_mute') === '1'; } catch (e) {}
    drawMute(); syncScreens();
  }
  function saveBest() { try { localStorage.setItem('starfold_best', String(best)); } catch (e) {} }
  function saveMute() { try { localStorage.setItem('starfold_mute', muted ? '1' : '0'); } catch (e) {} }

  function onTap(e) {
    initAudio();
    if (state === 'ready') newGame();
    else if (state === 'playing') flap();
    else if (state === 'dead' && (t - deadAt) > 0.45) newGame();
  }
  window.addEventListener('pointerdown', function (e) {
    if (e.target.closest && e.target.closest('#mute')) return;
    e.preventDefault(); onTap(e);
  }, { passive: false });
  window.addEventListener('keydown', function (e) {
    if (e.code === 'Space' || e.code === 'ArrowUp') { e.preventDefault(); onTap(e); }
  });

  const ON = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"/><path d="M19.07 4.93a10 10 0 0 1 0 14.14M15.54 8.46a5 5 0 0 1 0 7.07"/></svg>';
  const OFF = '<svg viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round"><polygon points="11 5 6 9 2 9 2 15 6 15 11 19 11 5"/><line x1="23" y1="9" x2="17" y2="15"/><line x1="17" y1="9" x2="23" y2="15"/></svg>';
  function drawMute() { muteBtn.innerHTML = muted ? OFF : ON; muteBtn.style.color = muted ? '#7a8f8b' : '#aef0e6'; }
  muteBtn.addEventListener('pointerdown', function (e) {
    e.stopPropagation(); e.preventDefault();
    muted = !muted; drawMute(); saveMute(); initAudio();
  });

  window.addEventListener('resize', resize);
  resize();
  drawMute();
  loadSaved();
  syncScreens();
  requestAnimationFrame(frame);
})();
</script>
</body>
</html>
```

---

## `app/src/main/AndroidManifest.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android">

    <uses-permission android:name="android.permission.INTERNET" />

    <application
        android:allowBackup="true"
        android:icon="@drawable/ic_launcher"
        android:roundIcon="@drawable/ic_launcher"
        android:label="STARFOLD"
        android:hardwareAccelerated="true"
        android:theme="@style/AppTheme">

        <activity
            android:name=".MainActivity"
            android:exported="true"
            android:screenOrientation="portrait"
            android:configChanges="orientation|screenSize|keyboardHidden|screenLayout">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>
    </application>
</manifest>
```

---

## `app/src/main/java/com/markfold/starfold/MainActivity.java`

```java
package com.markfold.starfold;

import android.app.Activity;
import android.os.Bundle;
import android.view.View;
import android.webkit.WebSettings;
import android.webkit.WebView;

public class MainActivity extends Activity {

    private WebView web;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);

        web = new WebView(this);
        WebSettings ws = web.getSettings();
        ws.setJavaScriptEnabled(true);
        ws.setDomStorageEnabled(true);            // enables localStorage
        ws.setMediaPlaybackRequiresUserGesture(false);
        web.setBackgroundColor(0xFF06080D);

        setContentView(web);
        web.loadUrl("file:///android_asset/index.html");
        hideSystemUi();
    }

    @Override
    public void onWindowFocusChanged(boolean hasFocus) {
        super.onWindowFocusChanged(hasFocus);
        if (hasFocus) hideSystemUi();
    }

    private void hideSystemUi() {
        getWindow().getDecorView().setSystemUiVisibility(
                View.SYSTEM_UI_FLAG_IMMERSIVE_STICKY
                        | View.SYSTEM_UI_FLAG_FULLSCREEN
                        | View.SYSTEM_UI_FLAG_HIDE_NAVIGATION
                        | View.SYSTEM_UI_FLAG_LAYOUT_STABLE
                        | View.SYSTEM_UI_FLAG_LAYOUT_FULLSCREEN
                        | View.SYSTEM_UI_FLAG_LAYOUT_HIDE_NAVIGATION);
    }

    @Override
    public void onBackPressed() {
        if (web != null && web.canGoBack()) web.goBack();
        else super.onBackPressed();
    }
}
```

---

## `app/src/main/res/values/themes.xml`

```xml
<?xml version="1.0" encoding="utf-8"?>
<resources>
    <style name="AppTheme" parent="@android:style/Theme.Material.NoActionBar">
        <item name="android:windowFullscreen">true</item>
        <item name="android:statusBarColor">#06080D</item>
        <item name="android:navigationBarColor">#06080D</item>
        <item name="android:windowLayoutInDisplayCutoutMode">shortEdges</item>
    </style>
</resources>
```

---

## `app/src/main/res/drawable/ic_launcher.xml`

Your real MarkFold mark, centered on an ink background, as a vector launcher icon.

```xml
<?xml version="1.0" encoding="utf-8"?>
<vector xmlns:android="http://schemas.android.com/apk/res/android"
    android:width="108dp"
    android:height="108dp"
    android:viewportWidth="108"
    android:viewportHeight="108">

    <path
        android:pathData="M16,0 L92,0 Q108,0 108,16 L108,92 Q108,108 92,108 L16,108 Q0,108 0,92 L0,16 Q0,0 16,0 Z"
        android:fillColor="#0A1018" />

    <group
        android:translateX="21"
        android:translateY="26.1"
        android:scaleX="0.109"
        android:scaleY="0.109">
        <group
            android:translateX="0"
            android:translateY="512"
            android:scaleX="0.1"
            android:scaleY="-0.1">
            <path android:fillColor="#FAC93D" android:pathData="M547 5106 c-3 -8 -30 -22 -59 -31 -124 -41 -231 -117 -326 -230 -48 -58 -103 -161 -117 -217 -4 -14 -15 -34 -26 -44 -19 -17 -19 -61 -19 -2001 0 -1961 0 -1983 20 -2003 11 -11 22 -32 26 -47 14 -60 81 -160 154 -233 66 -66 159 -140 177 -140 3 0 27 -13 52 -29 34 -22 72 -34 146 -47 55 -9 120 -20 144 -24 27 -4 69 -3 105 5 34 7 84 15 112 19 33 4 95 28 170 65 98 49 125 67 152 104 19 24 45 51 58 58 35 18 58 60 94 169 l31 95 -1 975 c0 634 3 987 10 1010 6 19 17 72 25 118 8 46 21 95 28 110 117 222 114 217 227 330 54 54 101 91 137 109 31 14 72 40 92 58 20 18 68 49 106 70 39 20 88 52 110 70 22 17 72 49 110 69 39 20 97 58 130 84 74 58 104 77 136 87 13 4 49 31 80 59 31 28 68 59 83 69 33 23 48 58 48 111 -1 53 -27 79 -177 180 -166 111 -213 141 -255 166 -19 11 -46 28 -60 39 -23 17 -180 119 -310 203 -133 85 -277 179 -299 195 -14 10 -54 36 -90 58 -91 56 -172 107 -302 193 -105 68 -197 115 -281 141 -20 7 -42 19 -48 27 -18 20 -385 21 -393 0z" />
            <path android:fillColor="#FAC93D" android:pathData="M5560 5004 c-42 -9 -129 -43 -158 -63 -15 -10 -70 -44 -122 -76 -52 -32 -112 -69 -132 -81 -21 -13 -57 -36 -80 -51 -24 -14 -59 -36 -78 -48 -19 -12 -69 -44 -110 -71 -41 -27 -86 -55 -100 -62 -14 -7 -32 -20 -41 -28 -8 -8 -19 -14 -23 -14 -4 0 -26 -13 -49 -28 -23 -16 -60 -40 -82 -54 -22 -13 -58 -36 -80 -50 -22 -13 -58 -36 -80 -50 -22 -13 -58 -36 -80 -50 -22 -14 -75 -47 -119 -74 -43 -27 -113 -72 -156 -101 -43 -28 -93 -60 -110 -70 -52 -29 -65 -37 -111 -68 -24 -15 -60 -38 -79 -50 -19 -12 -69 -44 -110 -71 -41 -27 -79 -51 -85 -53 -5 -2 -46 -27 -90 -56 -112 -72 -136 -87 -165 -105 -53 -31 -156 -97 -232 -147 -43 -29 -81 -53 -84 -53 -2 0 -17 -8 -32 -19 -48 -33 -230 -149 -322 -206 -70 -43 -126 -79 -167 -107 -23 -15 -45 -28 -51 -28 -5 0 -17 -9 -27 -20 -10 -11 -27 -23 -39 -26 -11 -4 -34 -16 -51 -28 -16 -12 -37 -26 -46 -31 -107 -67 -241 -193 -278 -261 -12 -21 -25 -46 -31 -54 -5 -8 -21 -42 -35 -75 -52 -117 -55 -169 -55 -940 0 -766 -1 -747 54 -773 24 -12 34 -12 70 2 23 9 60 33 82 53 21 20 68 51 104 69 36 18 88 51 115 72 28 22 100 68 160 103 61 34 124 76 142 94 17 18 51 42 75 54 24 11 68 39 98 60 30 22 82 56 116 76 33 19 69 44 79 55 9 10 39 26 66 35 26 9 63 32 82 51 36 37 191 134 214 134 7 0 28 15 46 33 19 18 67 53 108 78 41 25 88 54 105 65 55 34 218 142 259 172 22 17 60 40 85 52 25 12 74 44 110 70 36 25 88 59 115 75 28 15 61 38 75 50 29 27 117 75 136 75 8 0 22 8 32 17 9 10 41 33 70 52 29 18 60 44 70 56 10 13 49 37 86 54 37 17 77 40 89 51 49 45 161 120 192 129 19 5 60 31 93 57 68 56 139 103 296 194 62 36 139 86 171 111 33 25 87 61 120 78 33 18 85 56 115 85 30 29 76 69 103 90 26 20 47 43 47 50 0 7 22 42 48 78 36 47 51 77 55 112 4 25 13 59 22 75 12 25 15 99 15 473 0 475 -5 555 -40 633 -32 71 -133 166 -207 194 -59 23 -137 33 -183 25z" />
            <path android:fillColor="#FAC93D" android:pathData="M5505 2575 c-39 -19 -108 -60 -155 -90 -47 -30 -101 -64 -120 -75 -19 -11 -46 -28 -60 -38 -14 -11 -43 -30 -65 -44 -22 -14 -94 -60 -160 -103 -66 -43 -136 -88 -155 -100 -19 -12 -89 -57 -155 -100 -147 -96 -277 -180 -327 -212 -96 -59 -115 -71 -142 -89 -16 -10 -36 -24 -45 -29 -9 -6 -50 -32 -91 -59 -41 -27 -91 -59 -110 -71 -162 -102 -301 -198 -319 -221 -27 -34 -27 -84 0 -118 22 -28 41 -42 159 -118 36 -23 117 -75 180 -116 111 -72 147 -95 228 -145 20 -13 50 -32 66 -43 16 -10 37 -24 46 -29 10 -6 35 -22 56 -36 22 -14 74 -49 117 -77 44 -29 82 -52 87 -52 4 0 13 -7 21 -16 7 -8 29 -24 48 -35 20 -11 54 -32 77 -47 83 -55 150 -97 176 -112 15 -8 58 -32 94 -53 37 -21 89 -42 116 -47 28 -6 73 -17 101 -25 58 -17 134 -19 192 -5 22 6 69 14 104 19 40 6 71 17 85 30 12 12 43 28 68 37 29 9 67 36 102 70 31 30 74 65 96 79 22 14 40 31 40 38 0 15 46 104 71 136 10 12 21 39 25 60 3 20 13 60 21 89 12 44 14 160 11 770 l-4 717 -26 59 c-38 89 -121 175 -201 209 -96 39 -162 37 -252 -8z" />
        </group>
    </group>
</vector>
```

---

## `app/build.gradle`

```groovy
plugins {
    id 'com.android.application'
}

android {
    namespace 'com.markfold.starfold'
    compileSdk 34

    defaultConfig {
        applicationId "com.markfold.starfold"
        minSdk 24
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }

    buildTypes {
        release {
            minifyEnabled false
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
}
```

---

## `build.gradle` (project root)

```groovy
plugins {
    id 'com.android.application' version '8.5.2' apply false
}
```

---

## `settings.gradle`

```groovy
pluginManagement {
    repositories {
        google()
        mavenCentral()
        gradlePluginPortal()
    }
}
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
    }
}
rootProject.name = "STARFOLD"
include ':app'
```

---

## `gradle.properties`

```properties
org.gradle.jvmargs=-Xmx2048m -Dfile.encoding=UTF-8
android.useAndroidX=true
android.nonTransitiveRClass=true
```

---

## `.github/workflows/build.yml`

```yaml
name: Build STARFOLD APK

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: '17'

      - name: Set up Android SDK
        uses: android-actions/setup-android@v3

      - name: Generate Gradle wrapper
        run: gradle wrapper --gradle-version 8.7

      - name: Build debug APK
        run: |
          chmod +x ./gradlew
          ./gradlew assembleDebug --stacktrace

      - name: Upload APK
        uses: actions/upload-artifact@v4
        with:
          name: starfold-debug-apk
          path: app/build/outputs/apk/debug/app-debug.apk
```
