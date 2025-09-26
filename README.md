<!DOCTYPE html>
<html lang="fa" dir="rtl">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>FastLane — بازی ماشینی</title>
<style>
  :root{
    --bg:#071026; --panel:#0e1a2a; --accent:#ff6b6b; --accent2:#6ee7b7; --muted:#93a3b8;
  }
  *{box-sizing:border-box}
  body{
    margin:0; font-family: "Tahoma", "Segoe UI", Arial, sans-serif;
    background: linear-gradient(180deg,#021026 0%, #071026 100%);
    color:#e6eef8; display:flex; align-items:center; justify-content:center; min-height:100vh;
  }
  .wrap{width:100%;max-width:900px;padding:16px}
  .game {
    background: linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
    border-radius:14px; padding:12px; box-shadow:0 10px 30px rgba(0,0,0,0.6);
  }
  header{display:flex;justify-content:space-between;align-items:center;padding:8px 12px}
  h1{margin:0;font-size:20px;color:var(--accent)}
  .controls{display:flex;gap:8px;align-items:center}
  button{background:var(--panel);color:var(--muted);border:1px solid rgba(255,255,255,0.03);padding:8px 10px;border-radius:8px;cursor:pointer}
  button.primary{background:var(--accent);color:#081021;border:none;font-weight:700}
  #hud{display:flex;gap:12px;align-items:center;font-size:14px;color:var(--muted)}
  canvas{display:block;margin:12px auto;background:linear-gradient(180deg,#0b2433,#081824);border-radius:10px;width:100%;height:auto;touch-action:none}
  .footer{display:flex;justify-content:space-between;align-items:center;padding:8px 12px;color:var(--muted);font-size:13px}
  .overlay{position:absolute;left:0;right:0;top:0;bottom:0;display:flex;align-items:center;justify-content:center}
  .panel{background:rgba(4,8,14,0.7);padding:18px;border-radius:12px;text-align:center;max-width:520px}
  .panel h2{margin-top:0;color:var(--accent)}
  .row{margin-top:12px;display:flex;gap:8px;justify-content:center;flex-wrap:wrap}
  .touch-pad{position:fixed;left:0;bottom:0;height:200px;width:50%;opacity:.02}
  .touch-pad.right{right:0;left:auto}
  .small{font-size:12px;color:var(--muted)}
  .leaderboard{max-height:120px;overflow:auto;text-align:right;padding:8px;background:rgba(255,255,255,0.02);border-radius:8px}
  @media (max-width:600px){ canvas{height:70vh} .panel{margin:12px} }
</style>
</head>
<body>
<div class="wrap">
  <div class="game" id="gameRoot">
    <header>
      <h1>FastLane</h1>
      <div class="controls">
        <div id="hud">
          <span>امتیاز: <b id="score">0</b></span>
          <span>سطح: <b id="level">1</b></span>
          <span>سکه: <b id="coins">0</b></span>
        </div>
        <button id="btnPause">توقف</button>
        <button id="btnRestart" class="primary">شروع</button>
      </div>
    </header>

    <div style="position:relative">
      <canvas id="gameCanvas" width="720" height="900" aria-label="بازی FastLane"></canvas>

      <!-- پنل‌های تعاملی -->
      <div id="overlay" class="overlay" style="display:none">
        <div class="panel">
          <h2 id="overlayTitle">FastLane</h2>
          <p id="overlayText">آماده‌ای؟ راننده توی خط بمون و موانع رو دور بزن!</p>
          <div class="row">
            <button id="startBtn" class="primary">شروع بازی</button>
            <button id="resumeBtn">ادامه</button>
            <button id="settingsBtn">تنظیمات</button>
          </div>
          <div class="row small" style="margin-top:12px">
            <div class="leaderboard" id="leaderboardBox"></div>
          </div>
        </div>
      </div>
    </div>

    <div class="footer">
      <div class="small">صفحه‌کلید: ← → حرکت ، Space برای شتاب</div>
      <div class="small">نسخه: 1.0 — آفلاین</div>
    </div>
  </div>
</div>

<!-- نواحی لمسی برای موبایل -->
<div class="touch-pad left" id="padLeft"></div>
<div class="touch-pad right" id="padRight"></div>

<script>
/*
  FastLane — بازی ماشینی ساده اما کامل
  - همه چیز در یک فایل: ذخیره نمره، لمسی و کیبورد، صدا، اثرات ذرات
  - می‌تونی پارامترها رو در بخش CONFIG تغییر بدی
*/

// ---------- CONFIG ----------
const CONFIG = {
  canvasWidth: 720,
  canvasHeight: 900,
  laneCount: 3,
  player: { w: 48, h: 90, speed: 6, maxSpeed: 12 },
  spawnInterval: 1200, // ms
  coinSpawnInterval: 2500,
  difficultyIncreaseEvery: 15000, // ms
  maxObstacles: 6,
};

// ---------- STATE ----------
let canvas, ctx;
let lastTime = 0;
let running = false;
let paused = false;
let overlayVisible = true;
let score = 0;
let coins = 0;
let level = 1;
let speedMultiplier = 1;
let obstacles = [];
let coinsOnField = [];
let particles = [];
let keys = {};
let lastSpawn = 0;
let lastCoinSpawn = 0;
let lastDifficultyUp = 0;
let highScores = JSON.parse(localStorage.getItem('fastlane_scores') || '[]');

// ---------- AUDIO (simple beeps) ----------
const AudioCtx = window.AudioContext || window.webkitAudioContext;
let audioCtx = null;
function beep(freq=440, time=0.08, volume=0.1){
  try{
    if(!audioCtx) audioCtx = new AudioCtx();
    const o = audioCtx.createOscillator();
    const g = audioCtx.createGain();
    o.type = 'sine';
    o.frequency.value = freq;
    g.gain.value = volume;
    o.connect(g); g.connect(audioCtx.destination);
    o.start();
    o.stop(audioCtx.currentTime + time);
  }catch(e){ /*ignore*/ }
}

// ---------- UTIL ----------
function rand(min,max){ return Math.random()*(max-min)+min; }
function clamp(v,a,b){ return Math.max(a,Math.min(b,v)); }
function now(){ return performance.now(); }

// ---------- PLAYER ----------
const Player = {
  x: 0, y: 0, w: CONFIG.player.w, h: CONFIG.player.h, lane: 1, speed: CONFIG.player.speed,
  init(){
    this.lane = Math.floor(CONFIG.laneCount/2);
    this.w = CONFIG.player.w; this.h = CONFIG.player.h;
    this.x = laneToX(this.lane) - this.w/2;
    this.y = canvas.height - this.h - 40;
    this.speed = CONFIG.player.speed;
  },
  moveLeft(){ this.lane = clamp(this.lane-1, 0, CONFIG.laneCount-1); this.x = laneToX(this.lane)-this.w/2; },
  moveRight(){ this.lane = clamp(this.lane+1, 0, CONFIG.laneCount-1); this.x = laneToX(this.lane)-this.w/2; },
  accelerate(){ this.speed = clamp(this.speed+1, CONFIG.player.speed, CONFIG.player.maxSpeed); },
  decelerate(){ this.speed = clamp(this.speed-1, CONFIG.player.speed, CONFIG.player.maxSpeed); },
  draw(){
    // بدنه ماشین (شکل ساده)
    ctx.save();
    ctx.translate(this.x+this.w/2, this.y+this.h/2);
    // سایه
    ctx.fillStyle = 'rgba(0,0,0,0.25)'; ctx.fillRect(-this.w/2, this.h/2-6, this.w, 10);
    // بدنه
    ctx.fillStyle = '#ff6b6b'; ctx.strokeStyle = '#ff9a9a';
    roundRect(ctx, -this.w/2, -this.h/2, this.w, this.h, 8, true, true);
    // پنجره
    ctx.fillStyle = 'rgba(255,255,255,0.15)';
    roundRect(ctx, -this.w/2+8, -this.h/2+8, this.w-16, this.h/3, 5, true, false);
    ctx.restore();
  }
};

// ---------- LAYOUT HELPERS ----------
function laneToX(laneIndex){
  const pad = 60;
  const usable = canvas.width - pad*2;
  const laneW = usable / CONFIG.laneCount;
  return pad + laneW*laneIndex + laneW/2;
}

// ---------- OBSTACLES ----------
function spawnObstacle(){
  if(obstacles.length >= CONFIG.maxObstacles) return;
  const lane = Math.floor(rand(0, CONFIG.laneCount));
  const sizeW = rand(40, 80);
  const sizeH = rand(60, 110);
  const x = laneToX(lane) - sizeW/2;
  const y = -sizeH - rand(10, 200);
  const speed = rand(2.4, 4.0) * speedMultiplier;
  obstacles.push({ x, y, w:sizeW, h:sizeH, lane, speed, color: randomCarColor() });
}

function spawnCoin(){
  const lane = Math.floor(rand(0, CONFIG.laneCount));
  const size = 20;
  const x = laneToX(lane) - size/2;
  const y = -size - rand(50, 400);
  coinsOnField.push({ x,y,size,vy: rand(1.8,3.2) * speedMultiplier });
}

// ---------- PARTICLES ----------
function spawnParticles(x,y,count=12,color='#fff'){
  for(let i=0;i<count;i++){
    particles.push({
      x, y, vx: rand(-2.5,2.5
