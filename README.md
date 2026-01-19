<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1.0"/>
<title>Cobrinha Épica - Single & Multi 🐍🍔</title>
<style>
  * { margin:0; padding:0; box-sizing:border-box; }
  body {
    display:flex; flex-direction:column; justify-content:center; align-items:center;
    height:100vh; background:linear-gradient(135deg, #0f0f23, #1a1a2e, #16213e);
    color:white; font-family:'Segoe UI',Arial,sans-serif; user-select:none; overflow:hidden;
  }
  h1 { font-size:3.2rem; margin:20px; background:linear-gradient(90deg,lime,cyan); -webkit-background-clip:text; -webkit-text-fill-color:transparent; text-shadow:0 0 20px lime; }
  canvas { border:5px solid lime; background:#000; box-shadow:0 0 50px rgba(0,255,0,0.5); image-rendering:pixelated; touch-action:none; }
  #overlay {
    position:fixed; inset:0; background:rgba(0,0,0,0.92); backdrop-filter:blur(8px);
    display:flex; flex-direction:column; justify-content:center; align-items:center; z-index:1000;
  }
  button {
    font-size:1.6rem; padding:18px 50px; margin:18px; min-width:320px;
    border:none; border-radius:16px; font-weight:bold; cursor:pointer;
    background:linear-gradient(45deg,#00ff9d,#00d4ff); color:#000; box-shadow:0 0 25px #00ff9d;
    transition: all 0.3s ease;
  }
  button:hover { transform:scale(1.08); box-shadow:0 0 45px #00d4ff; }
  #hamburger {
    position:absolute; top:20px; right:30px; font-size:3.5rem; color:lime; cursor:pointer;
    text-shadow:0 0 15px lime; z-index:999; transition:0.3s;
  }
  #hamburger:hover { transform:scale(1.25); color:cyan; }
  #pauseMenu {
    position:absolute; top:100px; right:30px; background:rgba(0,0,0,0.9);
    border:3px solid lime; border-radius:16px; padding:25px; display:none; flex-direction:column;
    z-index:998; box-shadow:0 0 40px lime;
  }
  #pauseMenu button { font-size:1.4rem; padding:14px 40px; margin:10px 0; min-width:240px; }
</style>
</head>
<body>

<h1>🐍 COBRINHA ÉPICA 🐍</h1>
<canvas id="c" width="560" height="560"></canvas>

<div id="hamburger">≡</div>
<div id="pauseMenu">
  <button id="resume">▶ Continuar</button>
  <button id="restart">🔄 Reiniciar Modo</button>
  <button id="menu">🏠 Menu Principal</button>
</div>

<div id="overlay">
  <div id="modeSelect">
    <h1 style="font-size:4.5rem; margin-bottom:60px;">ESCOLHA O MODO</h1>
    <button id="single">Single Player (Clássico)</button>
    <button id="multi">Multiplayer Local (2 Jogadores)</button>
  </div>
  <div id="gameOver" style="display:none; text-align:center;">
    <h2 id="goTitle" style="font-size:5rem; margin:30px;">GAME OVER</h2>
    <div id="goMessage" style="font-size:2.2rem; margin:30px 0;"></div>
    <button id="playAgain">Jogar Novamente</button>
  </div>
</div>

<script>
// ==============================================
// CONFIGURAÇÕES GERAIS
// ==============================================
const canvas = document.getElementById('c');
const ctx = canvas.getContext('2d');
const grid = 20;
const cols = canvas.width / grid;
const rows = canvas.height / grid;

let mode = null;               // 'single' ou 'multi'
let state = 'menu';            // menu | playing | paused | over
let speed = 140;               // ms por tick (menor = mais rápido)

let audioCtx = null;
const initAudio = () => { audioCtx = new (window.AudioContext||window.webkitAudioContext)(); };
const resumeAudio = () => { if(audioCtx?.state==='suspended') audioCtx.resume(); };

// Sons simples
function beepEat() {
  if(!audioCtx) return;
  const o = audioCtx.createOscillator(), g = audioCtx.createGain();
  o.connect(g); g.connect(audioCtx.destination);
  o.frequency.value = 880; o.type='sine';
  g.gain.setValueAtTime(0.35, audioCtx.currentTime);
  g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime+0.18);
  o.start(); o.stop(audioCtx.currentTime+0.18);
}

function beepDie() {
  if(!audioCtx) return;
  const o = audioCtx.createOscillator(), g = audioCtx.createGain();
  o.connect(g); g.connect(audioCtx.destination);
  o.frequency.value = 160; o.type='sawtooth';
  g.gain.setValueAtTime(0.3, audioCtx.currentTime);
  g.gain.exponentialRampToValueAtTime(0.001, audioCtx.currentTime+0.7);
  o.start(); o.stop(audioCtx.currentTime+0.7);
}

// ==============================================
// SINGLE PLAYER
// ==============================================
const single = {
  snake: [],
  dx:1, dy:0,
  food: {},
  score:0,
  high: parseInt(localStorage.getItem('snakeHigh')||'0'),
  reset() {
    this.snake = [{x:Math.floor(cols/2),y:Math.floor(rows/2)}];
    this.dx=1; this.dy=0; this.score=0;
    this.spawnFood();
  },
  spawnFood() {
    do {
      this.food.x = Math.floor(Math.random()*cols);
      this.food.y = Math.floor(Math.random()*rows);
    } while(this.snake.some(s=>s.x===this.food.x && s.y===this.food.y));
  }
};

// ==============================================
// MULTIPLAYER
// ==============================================
const players = [
  { // P1 - Verde - Setas
    snake: [], dx:1, dy:0,
    colorHead:'#00ff41', colorBody:'#44ff77',
    food:{}, score:0, alive:true,
    spawnFood() {
      do {
        this.food.x = Math.floor(Math.random()*cols);
        this.food.y = Math.floor(Math.random()*rows);
      } while(players[0].snake.some(s=>s.x===this.food.x&&s.y===this.food.y)||
              players[1].snake.some(s=>s.x===this.food.x&&s.y===this.food.y));
    }
  },
  { // P2 - Azul - WASD
    snake: [], dx:-1, dy:0,
    colorHead:'#4169ff', colorBody:'#7799ff',
    food:{}, score:0, alive:true,
    spawnFood() { this.spawnFood = players[0].spawnFood; this.spawnFood(); }
  }
];

function resetMulti() {
  players[0].snake = [{x:Math.floor(cols*0.25),y:Math.floor(rows/2)}];
  players[0].dx=1; players[0].dy=0; players[0].score=0; players[0].alive=true;
  players[1].snake = [{x:Math.floor(cols*0.75),y:Math.floor(rows/2)}];
  players[1].dx=-1; players[1].dy=0; players[1].score=0; players[1].alive=true;
  players.forEach(p => p.spawnFood());
}

// ==============================================
// DESENHO - COBRA REALISTA + HAMBÚRGUER
// ==============================================
function drawBurger(x,y,color='lime') {
  const s=grid, cx=x*s+s/2, cy=y*s+s/2;
  ctx.save(); ctx.translate(cx,cy); ctx.shadowBlur=10; ctx.shadowColor=color;

  // Pão baixo
  ctx.fillStyle='#e8c39e'; ctx.beginPath(); ctx.roundRect(-s/2+4,s/5,s-8,s/2.5,12); ctx.fill();
  // Alface
  ctx.shadowBlur=0; ctx.fillStyle='#32cd32'; ctx.beginPath(); ctx.ellipse(0,s/4, s/3.5,6,0,0,Math.PI*2); ctx.fill();
  // Carne
  ctx.fillStyle='#5e2c04'; ctx.shadowColor='#3a1a00'; ctx.shadowBlur=6; ctx.beginPath(); ctx.arc(0,s/4,s/4,0,Math.PI*2); ctx.fill();
  // Pão alto
  ctx.shadowColor=color; ctx.shadowBlur=10; ctx.fillStyle='#e8c39e'; ctx.beginPath(); ctx.roundRect(-s/2+4,-s/2+s/5,s-8,s/2.5,12); ctx.fill();

  ctx.restore();
}

function drawSnakeHead(x,y,dx,dy,color) {
  const s=grid, angle = Math.atan2(dy,dx);
  ctx.save(); ctx.translate(x*s+s/2, y*s+s/2); ctx.rotate(angle);

  // Cabeça oval
  ctx.fillStyle=color; ctx.shadowColor=color; ctx.shadowBlur=15;
  ctx.beginPath(); ctx.ellipse(0,0,s*0.48,s*0.55,0,0,Math.PI*2); ctx.fill(); ctx.shadowBlur=0;

  // Olhos
  ctx.fillStyle='white'; ctx.beginPath();
  ctx.arc(-s*0.18, -s*0.15, s*0.09,0,Math.PI*2); ctx.fill();
  ctx.arc( s*0.18, -s*0.15, s*0.09,0,Math.PI*2); ctx.fill();
  ctx.fillStyle='black'; ctx.beginPath();
  ctx.arc(-s*0.14, -s*0.13, s*0.05,0,Math.PI*2); ctx.fill();
  ctx.arc( s*0.14, -s*0.13, s*0.05,0,Math.PI*2); ctx.fill();

  // Língua
  ctx.strokeStyle='#ff69b4'; ctx.lineWidth=3; ctx.beginPath();
  ctx.moveTo(0,s*0.35); ctx.lineTo(0,s*0.55); ctx.stroke();

  ctx.restore();
}

function draw() {
  ctx.fillStyle='#000'; ctx.fillRect(0,0,canvas.width,canvas.height);

  if(mode==='single') {
    drawBurger(single.food.x, single.food.y);
    single.snake.forEach((p,i)=>{
      if(i===0) drawSnakeHead(p.x,p.y,single.dx,single.dy,'#00ff41');
      else {
        ctx.fillStyle=`hsl(120,70%,${65-i*2}%)`; ctx.fillRect(p.x*grid+3,p.y*grid+3,grid-6,grid-6);
      }
    });

    ctx.fillStyle='white'; ctx.font='bold 32px Arial';
    ctx.fillText(`Pontos: ${single.score}`,30,50);
    ctx.fillText(`Recorde: ${single.high}`, canvas.width-30,50, 'right');
  }
  else { // multi
    drawBurger(players[0].food.x, players[0].food.y, 'lime');
    drawBurger(players[1].food.x, players[1].food.y, '#00d4ff');

    players.forEach(pl=>{
      pl.snake.forEach((p,i)=>{
        if(i===0) drawSnakeHead(p.x,p.y,pl.dx,pl.dy,pl.colorHead);
        else {
          ctx.fillStyle=pl.colorBody; ctx.fillRect(p.x*grid+3,p.y*grid+3,grid-6,grid-6);
        }
      });
    });

    ctx.font='bold 32px Arial';
    ctx.fillStyle='lime'; ctx.textAlign='left';  ctx.fillText(`J1 ${players[0].score}`,30,50);
    ctx.fillStyle='#00d4ff'; ctx.textAlign='right'; ctx.fillText(`${players[1].score} J2`,canvas.width-30,50);
  }
}

// ==============================================
// LÓGICA DO JOGO
// ==============================================
function update() {
  if(state!=='playing') return;

  if(mode==='single') {
    const h = {x:single.snake[0].x + single.dx, y:single.snake[0].y + single.dy};

    if(h.x<0||h.x>=cols||h.y<0||h.y>=rows||single.snake.some(s=>s.x===h.x&&s.y===h.y)){
      beepDie();
      if(single.score > single.high){ single.high=single.score; localStorage.setItem('snakeHigh',single.score); }
      document.getElementById('goTitle').textContent = 'GAME OVER';
      document.getElementById('goMessage').innerHTML = `Pontuação: <b>${single.score}</b><br>Recorde: <b>${single.high}</b>`;
      document.getElementById('gameOver').style.display='flex';
      state='over';
      return;
    }

    single.snake.unshift(h);
    if(h.x===single.food.x && h.y===single.food.y){
      single.score++; beepEat(); single.spawnFood();
      if(single.score%5===0) speed = Math.max(70,speed-8);
    } else single.snake.pop();
  }
  else { // multi
    players.forEach(p=>{
      if(!p.alive) return;
      const h = {x:p.snake[0].x + p.dx, y:p.snake[0].y + p.dy};

      if(h.x<0||h.x>=cols||h.y<0||h.y>=rows){
        p.alive=false; beepDie(); checkMultiEnd(); return;
      }

      if(players[0].snake.some(s=>s.x===h.x&&s.y===h.y)||
         players[1].snake.some(s=>s.x===h.x&&s.y===h.y)){
        p.alive=false; beepDie(); checkMultiEnd(); return;
      }

      p.snake.unshift(h);
      if(h.x===p.food.x && h.y===p.food.y){
        p.score++; beepEat(); p.spawnFood();
      } else p.snake.pop();
    });
  }
}

function checkMultiEnd(){
  if(!players[0].alive && !players[1].alive)
    document.getElementById('goMessage').textContent = 'Empate épico!';
  else if(!players[0].alive)
    document.getElementById('goMessage').innerHTML = '<span style="color:#00d4ff">Jogador 2 Venceu!</span>';
  else
    document.getElementById('goMessage').innerHTML = '<span style="color:lime">Jogador 1 Venceu!</span>';

  document.getElementById('goTitle').textContent = 'FIM DE PARTIDA';
  document.getElementById('gameOver').style.display='flex';
  state='over';
}

// ==============================================
// LOOP PRINCIPAL
// ==============================================
function loop(){
  if(state==='playing'){
    update();
    draw();
  }
  setTimeout(loop, speed);
}

// ==============================================
// CONTROLES
// ==============================================
document.addEventListener('keydown',e=>{
  if(state!=='playing') return;
  const k = e.key.toLowerCase();

  if(mode==='single'){
    if(k==='arrowup'    && single.dy!== 1){single.dx=0; single.dy=-1;}
    if(k==='arrowdown'  && single.dy!==-1){single.dx=0; single.dy= 1;}
    if(k==='arrowleft'  && single.dx!== 1){single.dx=-1;single.dy= 0;}
    if(k==='arrowright' && single.dx!==-1){single.dx= 1;single.dy= 0;}
  } else {
    // P1 setas
    if(k==='arrowup'    && players[0].dy!== 1){players[0].dx=0; players[0].dy=-1;}
    if(k==='arrowdown'  && players[0].dy!==-1){players[0].dx=0; players[0].dy= 1;}
    if(k==='arrowleft'  && players[0].dx!== 1){players[0].dx=-1;players[0].dy= 0;}
    if(k==='arrowright' && players[0].dx!==-1){players[0].dx= 1;players[0].dy= 0;}
    // P2 WASD
    if(k==='w' && players[1].dy!== 1){players[1].dx=0; players[1].dy=-1;}
    if(k==='s' && players[1].dy!==-1){players[1].dx=0; players[1].dy= 1;}
    if(k==='a' && players[1].dx!== 1){players[1].dx=-1;players[1].dy= 0;}
    if(k==='d' && players[1].dx!==-1){players[1].dx= 1;players[1].dy= 0;}
  }
});

// ==============================================
// MENU & EVENTOS
// ==============================================
document.getElementById('single').onclick = ()=>{
  resumeAudio(); initAudio();
  mode='single'; state='playing';
  document.getElementById('overlay').style.display='none';
  document.getElementById('hamburger').style.display='block';
  single.reset(); loop();
};

document.getElementById('multi').onclick = ()=>{
  resumeAudio(); initAudio();
  mode='multi'; state='playing';
  document.getElementById('overlay').style.display='none';
  document.getElementById('hamburger').style.display='block';
  resetMulti(); loop();
};

document.getElementById('playAgain').onclick = ()=>{
  document.getElementById('gameOver').style.display='none';
  document.getElementById('overlay').style.display='none';
  state='playing';
  if(mode==='single') single.reset();
  else resetMulti();
};

document.getElementById('hamburger').onclick =
document.getElementById('resume').onclick = ()=>{
  document.getElementById('pauseMenu').style.display = 
  document.getElementById('pauseMenu').style.display==='flex'?'none':'flex';
  state = state==='paused'?'playing':'paused';
};

document.getElementById('restart').onclick = ()=>{
  if(mode==='single') single.reset();
  else resetMulti();
  document.getElementById('pauseMenu').style.display='none';
  state='playing';
};

document.getElementById('menu').onclick = ()=>{
  document.getElementById('overlay').style.display='flex';
  document.getElementById('modeSelect').style.display='block';
  document.getElementById('gameOver').style.display='none';
  document.getElementById('pauseMenu').style.display='none';
  document.getElementById('hamburger').style.display='none';
  state='menu';
};

// Início
document.getElementById('overlay').style.display='flex';
loop();
</script>
</body>
</html>
