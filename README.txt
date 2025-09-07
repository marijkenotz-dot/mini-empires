<!doctype html>
<html lang="de">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<title>Mini Empires – Start & Kartenwahl</title>
<style>
:root{--bg:#121417;--panel:#1b1f24;--chip:#2a3037;--text:#e5e7eb}
html,body{margin:0;height:100%;background:var(--bg);color:var(--text);font-family:system-ui,-apple-system,Segoe UI,Roboto,Inter,sans-serif}
#hud{position:fixed;left:0;right:0;top:0;padding:8px;display:flex;gap:8px;flex-wrap:wrap;pointer-events:none;z-index:10}
.chip{pointer-events:auto;background:var(--chip);padding:7px 11px;border-radius:999px;box-shadow:0 2px 0 rgba(0,0,0,.25);font-weight:700}
#bar{position:fixed;left:0;right:0;bottom:0;display:flex;gap:8px;flex-wrap:wrap;justify-content:center;padding:10px;background:linear-gradient(180deg,transparent,rgba(0,0,0,.35) 30%, rgba(0,0,0,.6));z-index:10}
.btn{background:var(--panel);border:1px solid #2f3640;border-radius:14px;padding:8px 12px;font-weight:800;box-shadow:0 3px 0 rgba(0,0,0,.3);min-width:116px;text-align:center;color:var(--text)}
.mode{position:fixed;right:10px;bottom:84px;background:#0b3b2a;border:1px solid #14532d;border-radius:12px;padding:6px 10px;font-size:12px;font-weight:800;letter-spacing:.3px;z-index:10}
.wave{position:fixed;right:10px;top:10px;background:#4b1d1d;border:1px solid #7f1d1d;padding:6px 10px;border-radius:10px;font-weight:800;z-index:10}
.hint{position:fixed;left:10px;bottom:84px;opacity:.95;font-size:13px;color:#cbd5e1;z-index:10}
canvas{display:block;touch-action:none}

/* Start-Overlay */
#start{position:fixed;inset:0;background:linear-gradient(180deg,rgba(0,0,0,.6),rgba(0,0,0,.8));display:flex;align-items:center;justify-content:center;z-index:50}
.card{background:#0b1220;border:1px solid #1f2a44;border-radius:16px;box-shadow:0 8px 30px rgba(0,0,0,.5);padding:18px 18px 14px;max-width:620px;width:calc(100% - 32px)}
.card h1{margin:0 0 10px;font-size:20px}
.row{display:flex;gap:8px;flex-wrap:wrap;margin:8px 0}
.pill{border-radius:999px;padding:8px 14px;border:1px solid #334155;background:#172032;cursor:pointer;font-weight:800}
.pill.active{outline:2px solid #38bdf8}
.select{display:flex;gap:8px}
.small{font-size:12px;opacity:.85}
.startbtn{margin-top:10px;width:100%;padding:10px 14px;border-radius:12px;background:#1c3e1f;border:1px solid #2a6a2f;font-weight:900;letter-spacing:.3px}
</style>
</head>
<body>

<!-- HUD -->
<div id="hud" hidden>
  <div class="chip">🌲 Holz: <span id="wood">0</span></div>
  <div class="chip">🍞 Nahrung: <span id="food">150</span></div>
  <div class="chip">🪨 Stein: <span id="stone">0</span></div>
  <div class="chip">🪙 Gold: <span id="gold">0</span></div>
  <div class="chip">👥 Bev.: <span id="pop">1/8</span></div>
  <div class="chip">🎯 Ausgew.: <span id="sel">1</span></div>
</div>
<div class="wave" hidden>Welle: <span id="wave">0</span></div>
<div class="hint" hidden>Tippen: bewegen/auswählen • Ziehen: Mehrfachauswahl • Auf Ressource: sammeln • Doppeltipp: Stop</div>
<div class="mode" id="mode" hidden>Modus: Normal</div>

<!-- Spiel-Canvas -->
<canvas id="game" width="960" height="540"></canvas>

<!-- Action-Bar -->
<div id="bar" hidden>
  <button class="btn" id="spawnVillager">+ Dorfbewohner (50 🍞)</button>
  <button class="btn" id="trainMilitia">🛡️ Miliz (60 🍞)</button>
  <button class="btn" id="trainArcher">🏹 Bogenschütze (30 🍞 / 30 🪙)</button>
  <button class="btn" id="buildHouse">🏠 Haus (30 🌲)</button>
  <button class="btn" id="buildFarm">🌾 Farm (40 🌲)</button>
  <button class="btn" id="buildMill">🛖 Mühle (50 🌲)</button>
  <button class="btn" id="buildBarracks">🏰 Kaserne (60 🌲 / 20 🪨)</button>
  <button class="btn" id="pauseBtn">⏸ Pause</button>
</div>

<!-- Start-Overlay -->
<div id="start">
  <div class="card">
    <h1>Spiel starten</h1>
    <div class="small">1) Schwierigkeit wählen</div>
    <div class="row" id="row-diff">
      <div class="pill active" data-diff="leicht">Leicht</div>
      <div class="pill" data-diff="normal">Mittel</div>
      <div class="pill" data-diff="hart">Schwer</div>
      <div class="pill" data-diff="frieden" title="keine Wellen">Friedlich</div>
    </div>
    <div class="small">2) Karte wählen</div>
    <div class="row" id="row-map">
      <div class="pill active" data-map="wiese">Wiese (ausgewogen)</div>
      <div class="pill" data-map="wald">Waldreich (viel Holz)</div>
      <div class="pill" data-map="berge">Bergland (viel Stein/Gold)</div>
    </div>
    <button id="btn-start" class="startbtn">🎮 Spiel starten</button>
  </div>
</div>

<script>
/* --- Canvas, Helpers --- */
const cvs=document.getElementById('game'), ctx=cvs.getContext('2d'), W=cvs.width, H=cvs.height, TILE=48;
const $=id=>document.getElementById(id);
const r=(a,b)=>Math.random()*(b-a)+a, clamp=(v,a,b)=>Math.max(a,Math.min(b,v));
const dist=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);

/* --- UI refs --- */
const ui={wood:wood,food:food,stone:stone,gold:gold,pop:pop,sel:sel,wave:wave};
function showHUD(on){ for(const id of ['hud','bar','mode']) $(id).hidden=!on; $('.hint').hidden=!on; $('.wave').hidden=!on; }
function $(sel){return document.querySelector(sel.startsWith('#')?sel:('#'+sel));}

/* --- Game State --- */
let paused=false, wavesEnabled=true, difficulty='leicht', buildMode=null;
let trees=[],rocks=[],golds=[],berries=[],buildings=[],friendly=[],enemy=[],selected=new Set();
const res={wood:0,food:150,stone:0,gold:0,cap:8,wave:0};
const DIFFS={
  frieden:{first:1e9,every:1e9,waveSize:w=>0},
  leicht :{first:240,every:150,waveSize:w=>1+w},
  normal :{first:150,every:120,waveSize:w=>2+Math.floor(w*1.5)},
  hart   :{first: 90,every: 90,waveSize:w=>3+w*2}
};
let waveTimer=0;

/* --- Entities --- */
function addBuilding(type,x,y){
  const s={house:18,farm:20,mill:21,barracks:26,tc:30}[type];
  const hp={house:240,farm:240,mill:280,barracks:380,tc:560}[type];
  const drop={house:0,farm:0,mill:1,barracks:0,tc:1}[type];
  buildings.push({type,x,y,w:s*2,h:s*2,hp:hp,max:hp,dropoff:!!drop});
}
function villager(x,y){return{team:1,type:'villager',x,y,r:12,tx:x,ty:y,speed:120,task:null,target:null,_t:0,hp:60,max:60,atk:4,range:16,rate:1,last:0,carry:{type:null,amt:0,cap:10},remember:null}}
function militia(x,y){return{team:1,type:'militia',x,y,r:13,tx:x,ty:y,speed:135,task:null,target:null,_t:0,hp:90,max:90,atk:10,range:16,rate:0.75,last:0}}
function archer (x,y){return{team:1,type:'archer', x,y,r:12,tx:x,ty:y,speed:130,task:null,target:null,_t:0,hp:70,max:70,atk:8, range:85, rate:0.8,last:0}}
function raider (x,y){return{team:2,type:'raider', x,y,r:13,tx:x,ty:y,speed:115,task:'raid',target:null,_t:0,hp:65,max:65,atk:8, range:16, rate:0.9,last:0}}

function updateHUD(){
  ui.wood.textContent=res.wood; ui.food.textContent=res.food;
  ui.stone.textContent=res.stone; ui.gold.textContent=res.gold;
  ui.pop.textContent=`${friendly.length}/${res.cap}`; ui.sel.textContent=selected.size||0;
  ui.wave.textContent=res.wave;
}

/* --- Start Screen Logic --- */
let chosenDiff='leicht', chosenMap='wiese';
document.querySelectorAll('#row-diff .pill').forEach(p=>p.onclick=()=>{
  document.querySelectorAll('#row-diff .pill').forEach(x=>x.classList.remove('active'));
  p.classList.add('active'); chosenDiff=p.dataset.diff;
});
document.querySelectorAll('#row-map .pill').forEach(p=>p.onclick=()=>{
  document.querySelectorAll('#row-map .pill').forEach(x=>x.classList.remove('active'));
  p.classList.add('active'); chosenMap=p.dataset.map;
});
$('#btn-start').onclick=()=>startGame(chosenMap, chosenDiff);

function startGame(map, diff){
  difficulty=diff; wavesEnabled = diff!=='frieden';
  // reset state
  trees=[]; rocks=[]; golds=[]; berries=[]; buildings=[]; friendly=[]; enemy=[]; selected=new Set();
  Object.assign(res,{wood:0,food:150,stone:0,gold:0,cap:8,wave:0});
  waveTimer=0;

  // scatter by map
  const S={
    wiese :{t:22,r:12,g:10,b:10},
    wald  :{t:36,r:10,g:8 ,b:8 },
    berge :{t:16,r:18,g:16,b:8 }
  }[map]||{t:22,r:12,g:10,b:10};
  function scatter(list,n,rad,padTop=90){for(let i=0;i<n;i++){list.push({x:r(50,W-50),y:r(padTop,H-70),hp:rad*3,max:rad*3})}}
  scatter(trees,S.t,18); scatter(rocks,S.r,18); scatter(golds,S.g,18); scatter(berries,S.b,16);

  // Town Center + 3 villagers middle
  addBuilding('tc', W/2, H/2);
  friendly.push(villager(W/2-24, H/2-8), villager(W/2+24, H/2-8), villager(W/2, H/2+28));
  selected=new Set([friendly[0],friendly[1],friendly[2]]);
  updateHUD();

  // show UI, hide start
  document.getElementById('start').style.display='none';
  showHUD(true);
}

/* --- Input / Selection --- */
const pos=e=>{const b=cvs.getBoundingClientRect(); const t=e.touches?e.touches[0]:e; return {x:t.clientX-b.left,y:t.clientY-b.top}};
const pickUnit=(list,x,y)=>list.find(u=>Math.hypot(u.x-x,u.y-y)<u.r+8);
const pick=(list,x,y,rad)=>list.find(o=>Math.hypot(o.x-x,o.y-y)<rad && (o.hp===undefined||o.hp>0));
let lastTap=0,_mouse=null,dragStart=null,dragging=false;

function selectOne(u){selected.clear(); if(u) selected.add(u); updateHUD();}
function selectRect(x1,y1,x2,y2){
  const minx=Math.min(x1,x2),maxx=Math.max(x1,x2),miny=Math.min(y1,y2),maxy=Math.max(y1,y2);
  selected.clear(); for(const u of friendly) if(u.x>=minx&&u.x<=maxx&&u.y>=miny&&u.y<=maxy) selected.add(u);
  if(selected.size===0 && friendly[0]) selected.add(friendly[0]); updateHUD();
}
function setTask(u,kind,target){u.task=kind;u.target=target;u.tx=target.x;u.ty=target.y;u._t=0;}

function onDown(e){
  if($('#start').style.display!=='none') return; // still on start
  if(paused) return;
  const p=pos(e); _mouse=p; dragStart=p; dragging=false;
  const now=performance.now(), dbl=now-lastTap<250; lastTap=now;

  // Einheit auswählen?
  const hit=pickUnit(friendly,p.x,p.y);
  if(hit && !dbl && !buildMode){ selectOne(hit); return; }

  // Bauen?
  if(buildMode){
    if(placeBuilding(buildMode,p.x,p.y)) buildMode=null;
    return;
  }

  // Doppel-Tap → Stop
  if(dbl){ for(const u of selected){u.task=null;u.target=null;u.tx=u.x;u.ty=u.y;u.remember=null;} return; }

  // Ressourcen / Gegner
  const tTree=pick(trees,p.x,p.y,22), tRock=pick(rocks,p.x,p.y,22), tGold=pick(golds,p.x,p.y,22), tBerry=pick(berries,p.x,p.y,20);
  const fFarm=buildings.find(b=>b.type==='farm' && Math.hypot(b.x-p.x,b.y-p.y)<26);
  const foe=pickUnit(enemy,p.x,p.y);

  if(tTree||tRock||tGold||tBerry||fFarm){
    for(const u of selected){
      if(u.type!=='villager') continue;
      const tgt=tTree||tRock||tGold||tBerry||fFarm;
      const kind=tTree?'wood':tRock?'stone':tGold?'gold':tBerry?'food-node':'farm';
      setTask(u,kind,tgt); u.remember={kind,target:tgt};
    }
    return;
  }
  if(foe){ for(const u of selected){u.task='attack';u.target=foe;u.tx=foe.x;u.ty=foe.y;} return; }

  // Bewegung
  let i=0; for(const u of selected){const offx=((i%3)-1)*10,offy=(Math.floor(i/3)-1)*10;i++;u.task='move';u.target=null;u.tx=p.x+offx;u.ty=p.y+offy;u.remember=null;}
}
function onMove(e){ _mouse=pos(e); if(!dragStart) return; const dx=_mouse.x-dragStart.x, dy=_mouse.y-dragStart.y; dragging=Math.hypot(dx,dy)>12; }
function onUp(){ if(dragging && dragStart && _mouse) selectRect(dragStart.x,dragStart.y,_mouse.x,_mouse.y); dragStart=null; dragging=false; }
cvs.addEventListener('pointerdown',onDown,{passive:true});
cvs.addEventListener('pointermove',onMove,{passive:true});
cvs.addEventListener('pointerup',onUp,{passive:true});

/* --- Build/Buttons --- */
function canPlace(x,y){const far=50, ok=o=>Math.hypot(o.x-x,o.y-y)>far; return buildings.every(ok)&&trees.every(ok)&&rocks.every(ok)&&golds.every(ok)&&berries.every(ok);}
function costPay(c){for(const k in c) if((res[k]||0)<c[k]) return false; for(const k in c) res[k]-=c[k]; updateHUD(); return true;}
function placeBuilding(type,x,y){
  const costs={house:{wood:30}, farm:{wood:40}, mill:{wood:50}, barracks:{wood:60,stone:20}}[type]; if(!costs) return false;
  if(!canPlace(x,y)) return false; if(!costPay(costs)) return false;
  addBuilding(type,x,y); if(type==='house') res.cap+=3; updateHUD(); return true;
}
spawnVillager.onclick=()=>{ if(!buildings.some(b=>b.type==='tc'))return; if(res.food>=50 && friendly.length<res.cap){res.food-=50; friendly.push(villager(W/2+r(-60,60),H/2+r(-40,40))); updateHUD();} };
trainMilitia.onclick =()=>{ if(!buildings.some(b=>b.type==='barracks'))return; if(res.food>=60 && friendly.length<res.cap){res.food-=60; friendly.push(militia(W/2,H/2)); updateHUD();} };
trainArcher.onclick  =()=>{ if(!buildings.some(b=>b.type==='barracks'))return; if(res.food>=30 && res.gold>=30 && friendly.length<res.cap){res.food-=30; res.gold-=30; friendly.push(archer(W/2,H/2)); updateHUD();} };
buildHouse.onclick   =()=>{buildMode='house'; $('#mode').hidden=false; $('#mode').textContent='Modus: Bauen – Haus';};
buildFarm.onclick    =()=>{buildMode='farm';  $('#mode').hidden=false; $('#mode').textContent='Modus: Bauen – Farm';};
buildMill.onclick    =()=>{buildMode='mill';  $('#mode').hidden=false; $('#mode').textContent='Modus: Bauen – Mühle';};
buildBarracks.onclick=()=>{buildMode='barracks'; $('#mode').hidden=false; $('#mode').textContent='Modus: Bauen – Kaserne';};
pauseBtn.onclick     =()=>{paused=!paused; pauseBtn.textContent=paused?'▶ Weiter':'⏸ Pause';};

/* --- Economy helpers --- */
function nearestDropoff(obj){let best=null,bd=1e9; for(const b of buildings){if(b.dropoff){const d=Math.hypot(b.x-obj.x,b.y-obj.y); if(d<bd){bd=d;best=b;}}} return best;}
function farmsPassive(dt){for(const b of buildings){if(b.type==='farm'){b._p=(b._p||0)+dt; if(b._p>2){b._p=0; res.food+=1; updateHUD();}}}}

/* --- Combat helpers --- */
const nearestInSight=(u,list,sight)=>{let best=null,bd=1e9; for(const t of list){if(t.hp>0){const d=dist(u,t); if(d<bd && d<=sight){bd=d;best=t;}}} return best;}
const findNearestTarget=(u,list)=>{let best=null,bd=1e9; for(const t of list){if(t.hp>0){const d=dist(u,t); if(d<bd){bd=d;best=t;}}} return best;}
const damage=(obj,amt)=>{obj.hp=clamp(obj.hp-amt,0,obj.max)}

/* --- Waves --- */
function spawnWave(){
  const conf=DIFFS[difficulty], n=Math.max(0,Math.floor(conf.waveSize(res.wave)));
  if(n<=0) return; res.wave++;
  for(let i=0;i<n;i++){
    const edge=Math.floor(r(0,4)); let x,y;
    if(edge===0){x=-20;y=r(60,H-60);} if(edge===1){x=W+20;y=r(60,H-60);}
    if(edge===2){x=r(60,W-60);y=-20;} if(edge===3){x=r(60,W-60);y=H+20;}
    enemy.push(raider(x,y));
  } updateHUD();
}

/* --- Simulation loop --- */
function stepUnits(list,dt,isFriendly){
  for(const u of list){
    if(!isFriendly && (!u.target||u.target.hp<=0)){u.target=findNearestTarget(u,[...buildings,...friendly]); if(u.target){u.tx=u.target.x;u.ty=u.target.y;}}
    if(isFriendly && u.task==='attack' && u.target && u.target.hp<=0){u.task=null;}
    const dx=u.tx-u.x, dy=u.ty-u.y, d=Math.hypot(dx,dy); if(d>1){const s=u.speed*dt; u.x+=(dx/d)*Math.min(s,d); u.y+=(dy/d)*Math.min(s,d);}
    if(u.type==='villager'){
      if(u.task && d<=14){
        u._t+=dt; if(u._t>=0.5){u._t=0;
          if(u.task==='wood' && u.target?.hp>0){u.target.hp-=2;u.carry.type='wood';u.carry.amt+=2;}
          else if(u.task==='stone' && u.target?.hp>0){u.target.hp-=2;u.carry.type='stone';u.carry.amt+=2;}
          else if(u.task==='gold' && u.target?.hp>0){u.target.hp-=2;u.carry.type='gold';u.carry.amt+=2;}
          else if(u.task==='food-node' && u.target?.hp>0){u.target.hp-=2;u.carry.type='food';u.carry.amt+=2;}
          else if(u.task==='farm'){u.carry.type='food';u.carry.amt+=3;}
          if(u.carry.amt>=u.carry.cap){const dpo=nearestDropoff(u); if(dpo){u.task='return';u.target=dpo;u.tx=dpo.x;u.ty=dpo.y;}}
        }
      }
      if(u.task==='return' && d<=18){
        if(u.carry.type){res[u.carry.type]+=u.carry.amt; u.carry.amt=0; updateHUD();}
        if(u.remember && u.remember.target?.hp>0){setTask(u,u.remember.kind,u.remember.target);} else u.task=null;
      }
    }
    const foes=isFriendly?enemy:friendly;
    const tgt=(u.task==='attack'&&u.target)?u.target:nearestInSight(u,foes,u.type==='archer'?110:140);
    if(tgt){
      if(dist(u,tgt)<=u.range+2){u.last+=dt; if(u.last>=u.rate){u.last=0; damage(tgt,u.atk);} }
      else{const ang=Math.atan2(tgt.y-u.y,tgt.x-u.x); const keep=u.range-4; u.tx=tgt.x-Math.cos(ang)*keep; u.ty=tgt.y-Math.sin(ang)*keep;}
    }
  }
}
function cleanDead(){
  for(let i=friendly.length-1;i>=0;i--) if(friendly[i].hp<=0){friendly.splice(i,1);updateHUD();}
  for(let i=enemy.length-1;i>=0;i--) if(enemy[i].hp<=0) enemy.splice(i,1);
  for(let i=buildings.length-1;i>=0;i--) if(buildings[i].hp<=0) buildings.splice(i,1);
}

let last=0;
function loop(ts){
  const dt=Math.min(0.033,(ts-last)/1000); last=ts;
  if(!paused && $('#start').style.display==='none'){
    if(wavesEnabled){ const conf=DIFFS[difficulty]; waveTimer+=dt; const due=(res.wave===0)?conf.first:conf.every; if(waveTimer>=due){waveTimer=0; spawnWave();}}
    stepUnits(friendly,dt,true); stepUnits(enemy,dt,false); farmsPassive(dt); cleanDead();
  }
  draw(); requestAnimationFrame(loop);
}

/* --- Rendering --- */
function draw(){
  // simpler grüner Boden + bisschen Grain
  ctx.fillStyle='#1a2b1f'; ctx.fillRect(0,0,W,H);
  ctx.globalAlpha=.06; ctx.fillStyle='#fff'; for(let y=0;y<H;y+=TILE) for(let x=0;x<W;x+=TILE) ctx.fillRect(x,y,1,1); ctx.globalAlpha=1;

  if(dragging && dragStart && _mouse){
    ctx.fillStyle='rgba(59,130,246,.15)'; ctx.strokeStyle='rgba(59,130,246,.8)'; ctx.setLineDash([6,4]);
    const x=dragStart.x,y=dragStart.y,w=_mouse.x-x,h=_mouse.y-y; ctx.fillRect(x,y,w,h); ctx.strokeRect(x,y,w,h); ctx.setLineDash([]);
  }

  const blob=(x,y,rad,fill,stroke)=>{ctx.beginPath();ctx.arc(x,y,rad,0,Math.PI*2);ctx.fillStyle=fill;ctx.fill();ctx.lineWidth=2;ctx.strokeStyle=stroke;ctx.stroke();};
  for(const t of trees){if(t.hp>0){blob(t.x,t.y,18,'#2ecc71','#14532d'); ctx.fillStyle='#8e5a2c'; ctx.fillRect(t.x-2,t.y+12,4,10);}}
  for(const s of rocks){if(s.hp>0){blob(s.x,s.y,18,'#94a3b8','#334155');}}
  for(const g of golds){if(g.hp>0){blob(g.x,g.y,18,'#fbbf24','#92400e');}}
  for(const b of berries){if(b.hp>0){blob(b.x,b.y,16,'#b91c1c','#7f1d1d');}}

  for(const b of buildings){
    if(b.type==='house'){
      ctx.fillStyle='#9ca3af'; ctx.fillRect(b.x-18,b.y-18,36,36);
      ctx.fillStyle='#b91c1c'; ctx.beginPath(); ctx.moveTo(b.x-22,b.y-18); ctx.lineTo(b.x+22,b.y-18); ctx.lineTo(b.x,b.y-36); ctx.closePath(); ctx.fill();
      ctx.fillStyle='#1f2937'; ctx.fillRect(b.x-6,b.y+2,12,14);
    }else if(b.type==='farm'){
      ctx.fillStyle='#166534'; ctx.fillRect(b.x-20,b.y-20,40,40);
      ctx.fillStyle='rgba(255,255,255,.15)'; for(let i=-16;i<=16;i+=8) ctx.fillRect(b.x-20,b.y+i,40,2);
      ctx.strokeStyle='#34d399'; ctx.strokeRect(b.x-20,b.y-20,40,40);
    }else if(b.type==='mill'){
      ctx.fillStyle='#8b5cf6'; ctx.fillRect(b.x-21,b.y-21,42,42);
      ctx.fillStyle='#e5e7eb';
      ctx.beginPath(); ctx.moveTo(b.x,b.y-24); ctx.lineTo(b.x+6,b.y-6); ctx.lineTo(b.x-6,b.y-6); ctx.closePath(); ctx.fill();
      ctx.beginPath(); ctx.moveTo(b.x+24,b.y); ctx.lineTo(b.x+6,b.y+6); ctx.lineTo(b.x+6,b.y-6); ctx.closePath(); ctx.fill();
      ctx.beginPath(); ctx.moveTo(b.x,b.y+24); ctx.lineTo(b.x-6,b.y+6); ctx.lineTo(b.x+6,b.y+6); ctx.closePath(); ctx.fill();
      ctx.beginPath(); ctx.moveTo(b.x-24,b.y); ctx.lineTo(b.x-6,b.y-6); ctx.lineTo(b.x-6,b.y+6); ctx.closePath(); ctx.fill();
    }else if(b.type==='barracks'){
      ctx.fillStyle='#374151'; ctx.fillRect(b.x-26,b.y-26,52,52);
      ctx.fillStyle='#9ca3af'; ctx.fillRect(b.x-20,b.y-12,40,24);
      ctx.fillStyle='#e5e7eb'; ctx.fillRect(b.x-6,b.y-6,12,12);
      ctx.strokeStyle='#f59e0b'; ctx.setLineDash([6,4]); ctx.strokeRect(b.x-26,b.y-26,52,52); ctx.setLineDash([]);
    }else if(b.type==='tc'){
      ctx.fillStyle='#d1d5db'; ctx.fillRect(b.x-30,b.y-30,60,60);
      ctx.fillStyle='#92400e'; ctx.fillRect(b.x-30,b.y-30,60,10);
      ctx.fillStyle='#111827'; ctx.fillRect(b.x-8,b.y,16,18);
    }
    ctx.fillStyle='#0009'; ctx.fillRect(b.x-22,b.y-34,44,5);
    ctx.fillStyle='#22c55e'; ctx.fillRect(b.x-22,b.y-34,44*(b.hp/b.max),5);
  }

  function drawUnit(u,col)    .btn{background:var(--panel);border:1px solid #2f3640;border-radius:14px;padding:9px 13px;font-weight:800;box-shadow:0 3px 0 rgba(0,0,0,.3);min-width:120px;text-align:center}
    .hint{position:fixed;left:10px;bottom:84px;opacity:.95;font-size:13px;color:#cbd5e1;z-index:10}
    .mode{position:fixed;right:10px;bottom:84px;background:#0b3b2a;border:1px solid #14532d;border-radius:12px;padding:6px 10px;font-size:12px;font-weight:800;letter-spacing:.3px;z-index:10}
    .wave{position:fixed;right:10px;top:10px;background:#4b1d1d;border:1px solid #7f1d1d;padding:6px 10px;border-radius:10px;font-weight:800;z-index:10}
    canvas{display:block;touch-action:none}
  </style>
</head>
<body>
  <div id="hud">
    <div class="chip">🌲 Holz: <span id="wood">0</span></div>
    <div class="chip">🍞 Nahrung: <span id="food">150</span></div>
    <div class="chip">🪨 Stein: <span id="stone">0</span></div>
    <div class="chip">🪙 Gold: <span id="gold">0</span></div>
    <div class="chip">👥 Bev.: <span id="pop">1/8</span></div>
    <div class="chip">🎯 Ausgewählt: <span id="sel">1</span></div>
  </div>
  <div class="wave">Welle: <span id="wave">0</span></div>

  <canvas id="game" width="960" height="540"></canvas>

  <div class="hint">Tippen: bewegen/auswählen • Ziehen: Mehrfachauswahl • Auf Ressource: sammeln • Doppeltipp: Stop • Kaserne → Miliz/Bogenschütze</div>
  <div class="mode" id="mode">Modus: Normal</div>

  <div id="bar">
    <button class="btn" id="spawnVillager">+ Dorfbewohner (50 🍞)</button>
    <button class="btn" id="trainMilitia">🛡️ Miliz (60 🍞)</button>
    <button class="btn" id="trainArcher">🏹 Bogenschütze (30 🍞 / 30 🪙)</button>
    <button class="btn" id="buildHouse">🏠 Haus (30 🌲)</button>
    <button class="btn" id="buildFarm">🌾 Farm (40 🌲)</button>
    <button class="btn" id="buildMill">🛖 Mühle (50 🌲)</button>
    <button class="btn" id="buildBarracks">🏰 Kaserne (60 🌲 / 20 🪨)</button>
    <button class="btn" id="diffBtn">Modus: Leicht</button>
    <button class="btn" id="toggleWaves">Wellen: Ein</button>
    <button class="btn" id="pauseBtn">⏸ Pause</button>
  </div>

  <script>
  // === Setup =================================================================
  const cvs = document.getElementById('game'), ctx = cvs.getContext('2d');
  const W=cvs.width, H=cvs.height, TILE=48;
  let paused=false, wavesEnabled=true;

  // Ressourcen & UI
  const res = { wood:0, food:150, stone:0, gold:0, cap:8, wave:0 };
  const ui = {
    wood: wood, food: food, stone: stone, gold: gold, pop: pop, sel: sel,
    mode: document.getElementById('mode'), pause: document.getElementById('pauseBtn'),
    wave: document.getElementById('wave'), toggle: document.getElementById('toggleWaves'),
  };
  function updateHUD(){
    ui.wood.textContent=res.wood; ui.food.textContent=res.food;
    ui.stone.textContent=res.stone; ui.gold.textContent=res.gold;
    ui.pop.textContent=`${friendly.length}/${res.cap}`;
    ui.sel.textContent = selected.size || 0;
    ui.wave.textContent = res.wave;
  }

  // Schwierigkeitsgrade
  const DIFFS = {
    frieden: { first: 1e9,  every: 1e9,  waveSize: (w)=>0 },
    leicht:  { first: 240,  every: 150,  waveSize: (w)=> 1 + w },
    normal:  { first: 150,  every: 120,  waveSize: (w)=> 2 + Math.floor(w*1.5) },
    hart:    { first:  90,  every:  90,  waveSize: (w)=> 3 + w*2 }
  };
  const order = ['frieden','leicht','normal','hart'];
  let difficulty = 'leicht';
  let waveTimer = 0;

  // Helfer
  const r = (a,b)=>Math.random()*(b-a)+a;
  const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));
  const dist=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);

  // Ressourcen-Karte
  const trees=[], rocks=[], golds=[], berries=[];
  function scatter(list, n, radius, padTop=90){
    for(let i=0;i<n;i++){
      list.push({x:r(50,W-50), y:r(padTop,H-70), hp: radius*3, max: radius*3, kind:'res'});
    }
  }
  scatter(trees, 20, 18);
  scatter(rocks, 12, 18);
  scatter(golds, 10, 18);
  scatter(berries,10, 16);

  // Gebäude & Einheiten
  const buildings=[]; // {type,x,y,w,h,hp,max,dropoff?}
  const friendly=[];  // eigene Einheiten
  const enemy=[];     // Gegner

  // Start: Town Center (Drop-off) + 1 Villager
  function addBuilding(type,x,y){
    const sizes={house:18,farm:20,mill:21,barracks:26,tc:30};
    const hp   ={house:220,farm:220,mill:260,barracks:360,tc:520};
    const drop ={house:false,farm:false,mill:true,barracks:false,tc:true};
    const s=sizes[type];
    buildings.push({type,x,y,w:s*2,h:s*2,hp:hp[type],max:hp[type],dropoff:drop[type]});
  }
  addBuilding('tc', W/2, H/2); // Town Center
  function villager(x,y){return{team:1,type:'villager',x,y,r:12,tx:x,ty:y,speed:120,task:null,target:null,_t:0,hp:60,max:60,atk:4,range:16,rate:1,last:0,carry:{type:null,amt:0,cap:10},remember:null}}
  function militia(x,y){return{team:1,type:'militia', x,y,r:13,tx:x,ty:y,speed:135,task:null,target:null,_t:0,hp:90,max:90,atk:10,range:16,rate:0.75,last:0}}
  function archer (x,y){return{team:1,type:'archer',  x,y,r:12,tx:x,ty:y,speed:130,task:null,target:null,_t:0,hp:70,max:70,atk:8, range:85,rate:0.8,last:0}}
  function raider (x,y){return{team:2,type:'raider',  x,y,r:13,tx:x,ty:y,speed:115,task:'raid',target:null,_t:0,hp:65,max:65,atk:8, range:16,rate:0.9,last:0}}

  friendly.push( villager(W/2+40, H/2) );

  // Auswahl (Mehrfach)
  const selected = new Set(); // hält Referenzen auf Einheiten
  selectOne(friendly[0]);

  function selectOne(u){ selected.clear(); if(u) selected.add(u); updateHUD(); }
  function selectRect(x1,y1,x2,y2){
    const minx=Math.min(x1,x2), maxx=Math.max(x1,x2);
    const miny=Math.min(y1,y2), maxy=Math.max(y1,y2);
    selected.clear();
    for(const u of friendly){ if(u.x>=minx&&u.x<=maxx&&u.y>=miny&&u.y<=maxy) selected.add(u); }
    if(selected.size===0 && friendly[0]) selected.add(friendly[0]);
    updateHUD();
  }

  // Bau/Platzierung
  let buildMode=null; // 'house'|'farm'|'mill'|'barracks'
  function setMode(m){ buildMode=m; ui.mode.textContent = 'Modus: ' + (m? ('Bauen – '+m): 'Normal'); }
  function canPlace(x,y){
    const far=50;
    const ok = o => Math.hypot(o.x-x,o.y-y)>far;
    return buildings.every(ok) && trees.every(ok) && rocks.every(ok) && golds.every(ok) && berries.every(ok);
  }

  // Eingabe & Drag-Select
  function pos(e){ const b=cvs.getBoundingClientRect(); const x=(e.touches?e.touches[0].clientX:e.clientX)-b.left; const y=(e.touches?e.touches[0].clientY:e.clientY)-b.top; return {x,y}; }
  function pickUnit(list,x,y){ return list.find(u => Math.hypot(u.x-x,u.y-y) < u.r+8); }
  function pick(list,x,y,rad){ return list.find(o => Math.hypot(o.x-x,o.y-y) < rad && (o.hp===undefined || o.hp>0)); }

  let lastTap=0, _mouse=null;
  let dragStart=null, dragging=false;

  function onDown(e){
    if(paused) return;
    const p=pos(e);
    dragStart = p; dragging = false;

    const now=performance.now(); const dbl = now-lastTap<250; lastTap=now;

    // Einheit geklickt?
    const hit = pickUnit(friendly,p.x,p.y);
    if(hit && !dbl && !buildMode){ selectOne(hit); return; }

    // Bauen?
    if(buildMode){
      if(buildMode==='house'   && res.wood>=30 && canPlace(p.x,p.y)){ res.wood-=30; addBuilding('house',p.x,p.y); res.cap+=3; setMode(null); updateHUD(); return; }
      if(buildMode==='farm'    && res.wood>=40 && canPlace(p.x,p.y)){ res.wood-=40; addBuilding('farm',p.x,p.y); setMode(null); updateHUD(); return; }
      if(buildMode==='mill'    && res.wood>=50 && canPlace(p.x,p.y)){ res.wood-=50; addBuilding('mill',p.x,p.y); setMode(null); updateHUD(); return; }
      if(buildMode==='barracks'&& res.wood>=60 && res.stone>=20 && canPlace(p.x,p.y)){ res.wood-=60; res.stone-=20; addBuilding('barracks',p.x,p.y); setMode(null); updateHUD(); return; }
      return;
    }

    // Doppel-Tap → Stop (alle Selektierten)
    if(dbl){ for(const u of selected){ u.task=null; u.target=null; u.tx=u.x; u.ty=u.y; u.remember=null; } return; }

    // Ziele prüfen: Ressourcen / Gegner / Gebäude
    const tTree = pick(trees,p.x,p.y,22);
    const tRock = pick(rocks,p.x,p.y,22);
    const tGold = pick(golds,p.x,p.y,22);
    const tBerry= pick(berries,p.x,p.y,20);
    const fFarm = buildings.find(b=>b.type==='farm' && Math.hypot(b.x-p.x,b.y-p.y)<26);
    const foe    = pickUnit(enemy,p.x,p.y);

    if(tTree||tRock||tGold||tBerry||fFarm){
      for(const u of selected){
        if(u.type!=='villager') continue;
        const tgt = tTree||tRock||tGold||tBerry||fFarm;
        const kind = tTree?'wood':tRock?'stone':tGold?'gold':tBerry?'food-node':'farm';
        setTask(u,kind,tgt);
        u.remember = {kind,target:tgt}; // nach Abgabe erneut sammeln
      }
      return;
    }

    if(foe){
      for(const u of selected){ u.task='attack'; u.target=foe; u.tx=foe.x; u.ty=foe.y; }
      return;
    }

    // sonst: Bewegung (kleine Formationsoffsets)
    let i=0;
    for(const u of selected){
      const offx=((i%3)-1)*10, offy=(Math.floor(i/3)-1)*10; i++;
      u.task='move'; u.target=null; u.tx=p.x+offx; u.ty=p.y+offy;
      u.remember=null;
    }
  }
  function onMove(e){
    _mouse = pos(e);
    if(!dragStart) return;
    const dx=_mouse.x-dragStart.x, dy=_mouse.y-dragStart.y;
    dragging = Math.hypot(dx,dy) > 12;
  }
  function onUp(){
    if(dragging && dragStart && _mouse){ selectRect(dragStart.x,dragStart.y,_mouse.x,_mouse.y); }
    dragStart=null; dragging=false;
  }

  cvs.addEventListener('pointerdown', onDown, {passive:true});
  cvs.addEventListener('touchstart', onDown, {passive:true});
  cvs.addEventListener('pointermove', onMove, {passive:true});
  cvs.addEventListener('pointerup', onUp, {passive:true});
  cvs.addEventListener('pointerleave', ()=>{ _mouse=null; dragStart=null; dragging=false; }, {passive:true});

  function setTask(u,kind,target){ u.task=kind; u.target=target; u.tx=target.x; u.ty=target.y; u._t=0; }

  // Buttons
  spawnVillager.onclick=()=>{
    if(!buildings.some(b=>b.type==='tc')) return; // braucht TC
    if(res.food>=50 && friendly.length<res.cap){ res.food-=50; friendly.push(villager(W/2+ r(-60,60), H/2+r(-40,40))); updateHUD(); }
  };
  trainMilitia.onclick=()=>{
    if(!buildings.some(b=>b.type==='barracks')) return;
    if(res.food>=60 && friendly.length<res.cap){ res.food-=60; friendly.push(militia(W/2, H/2)); updateHUD(); }
  };
  trainArcher.onclick=()=>{
    if(!buildings.some(b=>b.type==='barracks')) return;
    if(res.food>=30 && res.gold>=30 && friendly.length<res.cap){ res.food-=30; res.gold-=30; friendly.push(archer(W/2, H/2)); updateHUD(); }
  };
  buildHouse.onclick   =()=> setMode('house');
  buildFarm.onclick    =()=> setMode('farm');
  buildMill.onclick    =()=> setMode('mill');
  buildBarracks.onclick=()=> setMode('barracks');
  pauseBtn.onclick     =()=>{ paused=!paused; pauseBtn.textContent = paused? '▶ Weiter':'⏸ Pause'; };
  diffBtn.onclick      =()=>{
    difficulty = order[(order.indexOf(difficulty)+1)%order.length];
    diffBtn.textContent = 'Modus: ' + (difficulty.charAt(0).toUpperCase()+difficulty.slice(1));
    waveTimer = 0; // Timer zurücksetzen beim Wechsel
  };
  toggleWaves.onclick  =()=>{
    wavesEnabled = !wavesEnabled;
    toggleWaves.textContent = wavesEnabled ? 'Wellen: Ein' : 'Wellen: Aus';
  };

  // Gegnerwellen
  function spawnWave(){
    const conf = DIFFS[difficulty];
    const n = Math.max(0, Math.floor(conf.waveSize(res.wave)));
    if(n<=0) return;
    res.wave++;
    for(let i=0;i<n;i++){
      const edge = Math.floor(r(0,4));
      let x,y;
      if(edge===0){ x=-20;    y=r(60,H-60); }
      if(edge===1){ x=W+20;   y=r(60,H-60); }
      if(edge===2){ x=r(60,W-60); y=-20; }
      if(edge===3){ x=r(60,W-60); y=H+20; }
      enemy.push( raider(x,y) );
    }
    updateHUD();
  }

  // === Loop ==================================================================
  let last=0;
  function loop(ts){
    const dt=Math.min(0.033,(ts-last)/1000); last=ts;
    if(!paused){
      // Wellen-Timer
      if(wavesEnabled){
        const conf = DIFFS[difficulty];
        waveTimer += dt;
        const due = (res.wave === 0) ? conf.first : conf.every;
        if (waveTimer >= due) { waveTimer = 0; spawnWave(); }
      }

      stepUnits(friendly, dt, true);
      stepUnits(enemy, dt, false);
      farmsPassive(dt);
      cleanDead();
    }
    draw();
    requestAnimationFrame(loop);
  }

  // Drop-off Helfer
  function nearestDropoff(obj){
    let best=null, bd=1e9;
    for(const b of buildings){ if(b.dropoff){ const d=Math.hypot(b.x-obj.x,b.y-obj.y); if(d<bd){ bd=d; best=b; } } }
    return best;
  }

  function stepUnits(list, dt, isFriendly){
    for(const u of list){
      // Auto-Ziel für Gegner
      if(!isFriendly && (!u.target || u.target.hp<=0)){
        u.target = findNearestTarget(u, [...buildings, ...friendly]);
        if(u.target){ u.tx=u.target.x; u.ty=u.target.y; }
      }
      if(isFriendly && (u.task==='attack') && u.target && u.target.hp<=0){ u.task=null; }

      // Bewegung
      const dx=u.tx-u.x, dy=u.ty-u.y, d=Math.hypot(dx,dy);
      if(d>1){ const s=u.speed*dt; u.x += (dx/d)*Math.min(s,d); u.y += (dy/d)*Math.min(s,d); }

      // Sammeln (Villager, mit Tragekapazität + Drop-off)
      if(u.type==='villager'){
        if(u.task && d<=14){
          u._t += dt;
          if(u._t>=0.5){
            u._t=0;
            if(u.task==='wood' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='wood'; u.carry.amt+=2; }
            else if(u.task==='stone' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='stone'; u.carry.amt+=2; }
            else if(u.task==='gold' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='gold'; u.carry.amt+=2; }
            else if(u.task==='food-node' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='food'; u.carry.amt+=2; }
            else if(u.task==='farm'){ u.carry.type='food'; u.carry.amt+=3; }

            // Kapazität erreicht → zum Drop-off
            if(u.carry.amt >= u.carry.cap){
              const dpo = nearestDropoff(u);
              if(dpo){ u.task='return'; u.target=dpo; u.tx=dpo.x; u.ty=dpo.y; }
            }
          }
        }
        // Abgabe
        if(u.task==='return' && d<=18){
          if(u.carry.type){ res[u.carry.type] += u.carry.amt; u.carry.amt=0; updateHUD(); }
          // zurück zur gemerkten Arbeit
          if(u.remember && u.remember.target?.hp>0){ setTask(u,u.remember.kind,u.remember.target); }
          else { u.task=null; }
        }
      }

      // Kampf
      const foes = isFriendly ? enemy : friendly;
      const tgt = (u.task==='attack' && u.target) ? u.target : nearestInSight(u, foes, u.type==='archer'? 110:150);
      if(tgt){
        if(dist(u,tgt)<=u.range+2){
          u.last += dt;
          if(u.last >= u.rate){ u.last = 0; damage(tgt, u.atk); }
        }else{
          // hinlaufen bis in Reichweite
          const ang = Math.atan2(tgt.y-u.y, tgt.x-u.x);
          const keep = u.type==='archer' ? u.range-6 : u.range-2;
          u.tx = tgt.x - Math.cos(ang)*keep;
          u.ty = tgt.y - Math.sin(ang)*keep;
        }
      }
    }
  }

  function nearestInSight(u, list, sight){
    let best=null, bd=1e9;
    for(const t of list){ if(t.hp>0){ const d=dist(u,t); if(d<bd && d<=sight){ bd=d; best=t; } } }
    return best;
  }
  function findNearestTarget(u, list){
    let best=null, bd=1e9;
    for(const t of list){ if(t.hp>0){ const d=dist(u,t); if(d<bd){ bd=d; best=t; } } }
    return best;
  }
  function damage(obj,amt){ obj.hp = clamp(obj.hp-amt, 0, obj.max); }

  function farmsPassive(dt){
    for(const b of buildings){
      if(b.type==='farm'){
        b._passive=(b._passive||0)+dt;
        if(b._passive>2){ b._passive=0; res.food+=1; updateHUD(); }
      }
    }
  }
  function cleanDead(){
    for(let i=friendly.length-1;i>=0;i--) if(friendly[i].hp<=0){ friendly.splice(i,1); updateHUD(); }
    for(let i=enemy.length-1;i>=0;i--) if(enemy[i].hp<=0){ enemy.splice(i,1); }
    for(let i=buildings.length-1;i>=0;i--) if(buildings[i].hp<=0){ buildings.splice(i,1); }
  }

  // === Render ================================================================
  function hpbar(x,y,w,h,p,c){
    ctx.fillStyle='#000a'; ctx.fillRect(x-w/2,y,w,h);
    ctx.fillStyle=c; ctx.fillRect(x-w/2,y,w*p,h);
    ctx.strokeStyle='#000'; ctx.strokeRect(x-w/2,y,w,h);
  }
  function draw(){
    // Wiese + Grid
    ctx.fillStyle='#16321c'; ctx.fillRect(0,0,W,H);
    ctx.globalAlpha=.06; ctx.fillStyle='#fff';
    for(let y=0;y<H;y+=TILE) for(let x=0;x<W;x+=TILE) ctx.fillRect(x,y,1,1);
    ctx.globalAlpha=1;

    // Auswahl-Rechteck
    if(dragging && dragStart && _mouse){
      ctx.fillStyle='rgba(59,130,246,.15)'; ctx.strokeStyle='rgba(59,130,246,.8)'; ctx.setLineDash([6,4]);
      const x=dragStart.x, y=dragStart.y, w=_mouse.x-x, h=_mouse.y-y;
      ctx.fillRect(x,y,w,h); ctx.strokeRect(x,y,w,h); ctx.setLineDash([]);
    }

    // Ressourcen
    const blob=(x,y,rad,fill,stroke)=>{ctx.beginPath();ctx.arc(x,y,rad,0,Math.PI*2);ctx.fillStyle=fill;ctx.fill();ctx.lineWidth=2;ctx.strokeStyle=stroke;ctx.stroke();};
    for(const t of trees){ if(t.hp>0){ blob(t.x,t.y,18,'#2ecc71','#14532d'); ctx.fillStyle='#8e5a2c'; ctx.fillRect(t.x-2,t.y+12,4,10);} }
    for(const s of rocks){ if(s.hp>0){ blob(s.x,s.y,18,'#94a3b8','#334155'); } }
    for(const g of golds){ if(g.hp>0){ blob(g.x,g.y,18,'#fbbf24','#92400e'); } }
    for(const b of berries){ if(b.hp>0){ blob(b.x,b.y,16,'#b91c1c','#7f1d1d'); } }

    // Gebäude
    for(const b of buildings){
      if(b.type==='house'){
        ctx.fillStyle='#9ca3af'; ctx.fillRect(b.x-18,b.y-18,36,36);
        ctx.fillStyle='#b91c1c'; ctx.beginPath(); ctx.moveTo(b.x-22,b.y-18); ctx.lineTo(b.x+22,b.y-18); ctx.lineTo(b.x,b.y-36); ctx.closePath(); ctx.fill();
        ctx.fillStyle='#1f2937'; ctx.fillRect(b.x-6,b.y+2,12,14);
      }else if(b.type==='farm'){
        ctx.fillStyle='#166534'; ctx.fillRect(b.x-20,b.y-20,40,40);
        ctx.fillStyle='rgba(255,255,255,.15)'; for(let i=-16;i<=16;i+=8){ ctx.fillRect(b.x-20,b.y+i,40,2); }
        ctx.strokeStyle='#34d399'; ctx.strokeRect(b.x-20,b.y-20,40,40);
      }else if(b.type==='mill'){
        ctx.fillStyle='#8b5cf6'; ctx.fillRect(b.x-21,b.y-21,42,42);
        ctx.fillStyle='#e5e7eb';
        ctx.beginPath(); ctx.moveTo(b.x,b.y-24); ctx.lineTo(b.x+6,b.y-6); ctx.lineTo(b.x-6,b.y-6); ctx.closePath(); ctx.fill();
        ctx.beginPath(); ctx.moveTo(b.x+24,b.y); ctx.lineTo(b.x+6,b.y+6); ctx.lineTo(b.x+6,b.y-6); ctx.closePath(); ctx.fill();
        ctx.beginPath(); ctx.moveTo(b.x,b.y+24); ctx.lineTo(b.x-6,b.y+6); ctx.lineTo(b.x+6,b.y+6); ctx.closePath(); ctx.fill();
        ctx.beginPath(); ctx.moveTo(b.x-24,b.y); ctx.lineTo(b.x-6,b.y-6); ctx.lineTo(b.x-6,b.y+6); ctx.closePath(); ctx.fill();
      }else if(b.type==='barracks'){
        ctx.fillStyle='#374151'; ctx.fillRect(b.x-26,b.y-26,52,52);
        ctx.fillStyle='#9ca3af'; ctx.fillRect(b.x-20,b.y-12,40,24);
        ctx.fillStyle='#e5e7eb'; ctx.fillRect(b.x-6,b.y-6,12,12);
        ctx.strokeStyle='#f59e0b'; ctx.setLineDash([6,4]); ctx.strokeRect(b.x-26,b.y-26,52,52); ctx.setLineDash([]);
      }else    .btn{background:var(--panel);border:1px solid #2f3640;border-radius:14px;padding:9px 13px;font-weight:800;box-shadow:0 3px 0 rgba(0,0,0,.3);min-width:120px;text-align:center}
    .hint{position:fixed;left:10px;bottom:84px;opacity:.95;font-size:13px;color:#cbd5e1;z-index:10}
    .mode{position:fixed;right:10px;bottom:84px;background:#0b3b2a;border:1px solid #14532d;border-radius:12px;padding:6px 10px;font-size:12px;font-weight:800;letter-spacing:.3px;z-index:10}
    .wave{position:fixed;right:10px;top:10px;background:#4b1d1d;border:1px solid #7f1d1d;padding:6px 10px;border-radius:10px;font-weight:800;z-index:10}
    canvas{display:block;touch-action:none}
  </style>
</head>
<body>
  <div id="hud">
    <div class="chip">🌲 Holz: <span id="wood">0</span></div>
    <div class="chip">🍞 Nahrung: <span id="food">150</span></div>
    <div class="chip">🪨 Stein: <span id="stone">0</span></div>
    <div class="chip">🪙 Gold: <span id="gold">0</span></div>
    <div class="chip">👥 Bev.: <span id="pop">1/8</span></div>
    <div class="chip">🎯 Ausgewählt: <span id="sel">1</span></div>
  </div>
  <div class="wave">Welle: <span id="wave">0</span></div>

  <canvas id="game" width="960" height="540"></canvas>

  <div class="hint">Tippen: bewegen/auswählen • Ziehen: Mehrfachauswahl • Auf Ressource: sammeln • Doppeltipp: Stop • Kaserne → Miliz/Bogenschütze</div>
  <div class="mode" id="mode">Modus: Normal</div>

  <div id="bar">
    <button class="btn" id="spawnVillager">+ Dorfbewohner (50 🍞)</button>
    <button class="btn" id="trainMilitia">🛡️ Miliz (60 🍞)</button>
    <button class="btn" id="trainArcher">🏹 Bogenschütze (30 🍞 / 30 🪙)</button>
    <button class="btn" id="buildHouse">🏠 Haus (30 🌲)</button>
    <button class="btn" id="buildFarm">🌾 Farm (40 🌲)</button>
    <button class="btn" id="buildMill">🛖 Mühle (50 🌲)</button>
    <button class="btn" id="buildBarracks">🏰 Kaserne (60 🌲 / 20 🪨)</button>
    <button class="btn" id="diffBtn">Modus: Leicht</button>
    <button class="btn" id="toggleWaves">Wellen: Ein</button>
    <button class="btn" id="pauseBtn">⏸ Pause</button>
  </div>

  <script>
  // === Setup =================================================================
  const cvs = document.getElementById('game'), ctx = cvs.getContext('2d');
  const W=cvs.width, H=cvs.height, TILE=48;
  let paused=false, wavesEnabled=true;

  // Ressourcen & UI
  const res = { wood:0, food:150, stone:0, gold:0, cap:8, wave:0 };
  const ui = {
    wood: wood, food: food, stone: stone, gold: gold, pop: pop, sel: sel,
    mode: document.getElementById('mode'), pause: document.getElementById('pauseBtn'),
    wave: document.getElementById('wave'), toggle: document.getElementById('toggleWaves'),
  };
  function updateHUD(){
    ui.wood.textContent=res.wood; ui.food.textContent=res.food;
    ui.stone.textContent=res.stone; ui.gold.textContent=res.gold;
    ui.pop.textContent=`${friendly.length}/${res.cap}`;
    ui.sel.textContent = selected.size || 0;
    ui.wave.textContent = res.wave;
  }

  // Schwierigkeitsgrade
  const DIFFS = {
    frieden: { first: 1e9,  every: 1e9,  waveSize: (w)=>0 },
    leicht:  { first: 240,  every: 150,  waveSize: (w)=> 1 + w },         // 4:00, dann 2:30
    normal:  { first: 150,  every: 120,  waveSize: (w)=> 2 + Math.floor(w*1.5) },
    hart:    { first:  90,  every:  90,  waveSize: (w)=> 3 + w*2 }
  };
  const order = ['frieden','leicht','normal','hart'];
  let difficulty = 'leicht';
  let waveTimer = 0;

  // Helfer
  const r = (a,b)=>Math.random()*(b-a)+a;
  const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));
  const dist=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);

  // Ressourcen-Karte
  const trees=[], rocks=[], golds=[], berries=[];
  function scatter(list, n, radius, padTop=90){
    for(let i=0;i<n;i++){
      list.push({x:r(50,W-50), y:r(padTop,H-70), hp: radius*3, max: radius*3, kind:'res'});
    }
  }
  scatter(trees, 20, 18);
  scatter(rocks, 12, 18);
  scatter(golds, 10, 18);
  scatter(berries,10, 16);

  // Gebäude & Einheiten
  const buildings=[]; // {type,x,y,w,h,hp,max,dropoff?}
  const friendly=[];  // eigene Einheiten
  const enemy=[];     // Gegner

  // Start: Town Center (Drop-off) + 1 Villager
  function addBuilding(type,x,y){
    const sizes={house:18,farm:20,mill:21,barracks:26,tc:30};
    const hp   ={house:220,farm:220,mill:260,barracks:360,tc:520};
    const drop ={house:false,farm:false,mill:true,barracks:false,tc:true};
    const s=sizes[type];
    buildings.push({type,x,y,w:s*2,h:s*2,hp:hp[type],max:hp[type],dropoff:drop[type]});
  }
  addBuilding('tc', W/2, H/2); // Town Center
  function villager(x,y){return{team:1,type:'villager',x,y,r:12,tx:x,ty:y,speed:120,task:null,target:null,_t:0,hp:60,max:60,atk:4,range:16,rate:1,last:0,carry:{type:null,amt:0,cap:10},remember:null}}
  function militia(x,y){return{team:1,type:'militia', x,y,r:13,tx:x,ty:y,speed:135,task:null,target:null,_t:0,hp:90,max:90,atk:10,range:16,rate:0.75,last:0}}
  function archer (x,y){return{team:1,type:'archer',  x,y,r:12,tx:x,ty:y,speed:130,task:null,target:null,_t:0,hp:70,max:70,atk:8, range:85,rate:0.8,last:0}}
  function raider (x,y){return{team:2,type:'raider',  x,y,r:13,tx:x,ty:y,speed:115,task:'raid',target:null,_t:0,hp:65,max:65,atk:8, range:16,rate:0.9,last:0}}

  friendly.push( villager(W/2+40, H/2) );

  // Auswahl (Mehrfach)
  const selected = new Set(); // hält Referenzen auf Einheiten
  selectOne(friendly[0]);

  function selectOne(u){ selected.clear(); if(u) selected.add(u); updateHUD(); }
  function selectRect(x1,y1,x2,y2){
    const minx=Math.min(x1,x2), maxx=Math.max(x1,x2);
    const miny=Math.min(y1,y2), maxy=Math.max(y1,y2);
    selected.clear();
    for(const u of friendly){ if(u.x>=minx&&u.x<=maxx&&u.y>=miny&&u.y<=maxy) selected.add(u); }
    if(selected.size===0 && friendly[0]) selected.add(friendly[0]);
    updateHUD();
  }

  // Bau/Platzierung
  let buildMode=null; // 'house'|'farm'|'mill'|'barracks'
  function setMode(m){ buildMode=m; ui.mode.textContent = 'Modus: ' + (m? ('Bauen – '+m): 'Normal'); }
  function canPlace(x,y){
    const far=50;
    const ok = o => Math.hypot(o.x-x,o.y-y)>far;
    return buildings.every(ok) && trees.every(ok) && rocks.every(ok) && golds.every(ok) && berries.every(ok);
  }

  // Eingabe & Drag-Select
  function pos(e){ const b=cvs.getBoundingClientRect(); const x=(e.touches?e.touches[0].clientX:e.clientX)-b.left; const y=(e.touches?e.touches[0].clientY:e.clientY)-b.top; return {x,y}; }
  function pickUnit(list,x,y){ return list.find(u => Math.hypot(u.x-x,u.y-y) < u.r+8); }
  function pick(list,x,y,rad){ return list.find(o => Math.hypot(o.x-x,o.y-y) < rad && (o.hp===undefined || o.hp>0)); }

  let lastTap=0, _mouse=null;
  let dragStart=null, dragging=false;

  function onDown(e){
    if(paused) return;
    const p=pos(e);
    dragStart = p; dragging = false;

    const now=performance.now(); const dbl = now-lastTap<250; lastTap=now;

    // Einheit geklickt?
    const hit = pickUnit(friendly,p.x,p.y);
    if(hit && !dbl && !buildMode){ selectOne(hit); return; }

    // Bauen?
    if(buildMode){
      if(buildMode==='house'   && res.wood>=30 && canPlace(p.x,p.y)){ res.wood-=30; addBuilding('house',p.x,p.y); res.cap+=3; setMode(null); updateHUD(); return; }
      if(buildMode==='farm'    && res.wood>=40 && canPlace(p.x,p.y)){ res.wood-=40; addBuilding('farm',p.x,p.y); setMode(null); updateHUD(); return; }
      if(buildMode==='mill'    && res.wood>=50 && canPlace(p.x,p.y)){ res.wood-=50; addBuilding('mill',p.x,p.y); setMode(null); updateHUD(); return; }
      if(buildMode==='barracks'&& res.wood>=60 && res.stone>=20 && canPlace(p.x,p.y)){ res.wood-=60; res.stone-=20; addBuilding('barracks',p.x,p.y); setMode(null); updateHUD(); return; }
      return;
    }

    // Doppel-Tap → Stop (alle Selektierten)
    if(dbl){ for(const u of selected){ u.task=null; u.target=null; u.tx=u.x; u.ty=u.y; u.remember=null; } return; }

    // Ziele prüfen: Ressourcen / Gegner / Gebäude
    const tTree = pick(trees,p.x,p.y,22);
    const tRock = pick(rocks,p.x,p.y,22);
    const tGold = pick(golds,p.x,p.y,22);
    const tBerry= pick(berries,p.x,p.y,20);
    const fFarm = buildings.find(b=>b.type==='farm' && Math.hypot(b.x-p.x,b.y-p.y)<26);
    const foe    = pickUnit(enemy,p.x,p.y);

    if(tTree||tRock||tGold||tBerry||fFarm){
      for(const u of selected){
        if(u.type!=='villager') continue;
        const tgt = tTree||tRock||tGold||tBerry||fFarm;
        const kind = tTree?'wood':tRock?'stone':tGold?'gold':tBerry?'food-node':'farm';
        setTask(u,kind,tgt);
        u.remember = {kind,target:tgt}; // nach Abgabe erneut sammeln
      }
      return;
    }

    if(foe){
      for(const u of selected){ u.task='attack'; u.target=foe; u.tx=foe.x; u.ty=foe.y; }
      return;
    }

    // sonst: Bewegung (kleine Formationsoffsets)
    let i=0;
    for(const u of selected){
      const offx=((i%3)-1)*10, offy=(Math.floor(i/3)-1)*10; i++;
      u.task='move'; u.target=null; u.tx=p.x+offx; u.ty=p.y+offy;
      u.remember=null;
    }
  }
  function onMove(e){
    _mouse = pos(e);
    if(!dragStart) return;
    const dx=_mouse.x-dragStart.x, dy=_mouse.y-dragStart.y;
    dragging = Math.hypot(dx,dy) > 12;
  }
  function onUp(){
    if(dragging && dragStart && _mouse){ selectRect(dragStart.x,dragStart.y,_mouse.x,_mouse.y); }
    dragStart=null; dragging=false;
  }

  cvs.addEventListener('pointerdown', onDown, {passive:true});
  cvs.addEventListener('touchstart', onDown, {passive:true});
  cvs.addEventListener('pointermove', onMove, {passive:true});
  cvs.addEventListener('pointerup', onUp, {passive:true});
  cvs.addEventListener('pointerleave', ()=>{ _mouse=null; dragStart=null; dragging=false; }, {passive:true});

  function setTask(u,kind,target){ u.task=kind; u.target=target; u.tx=target.x; u.ty=target.y; u._t=0; }

  // Buttons
  spawnVillager.onclick=()=>{
    if(!buildings.some(b=>b.type==='tc')) return; // braucht TC
    if(res.food>=50 && friendly.length<res.cap){ res.food-=50; friendly.push(villager(W/2+ r(-60,60), H/2+r(-40,40))); updateHUD(); }
  };
  trainMilitia.onclick=()=>{
    if(!buildings.some(b=>b.type==='barracks')) return;
    if(res.food>=60 && friendly.length<res.cap){ res.food-=60; friendly.push(militia(W/2, H/2)); updateHUD(); }
  };
  trainArcher.onclick=()=>{
    if(!buildings.some(b=>b.type==='barracks')) return;
    if(res.food>=30 && res.gold>=30 && friendly.length<res.cap){ res.food-=30; res.gold-=30; friendly.push(archer(W/2, H/2)); updateHUD(); }
  };
  buildHouse.onclick   =()=> setMode('house');
  buildFarm.onclick    =()=> setMode('farm');
  buildMill.onclick    =()=> setMode('mill');
  buildBarracks.onclick=()=> setMode('barracks');
  pauseBtn.onclick     =()=>{ paused=!paused; pauseBtn.textContent = paused? '▶ Weiter':'⏸ Pause'; };
  diffBtn.onclick      =()=>{
    difficulty = order[(order.indexOf(difficulty)+1)%order.length];
    diffBtn.textContent = 'Modus: ' + (difficulty.charAt(0).toUpperCase()+difficulty.slice(1));
    waveTimer = 0; // Timer zurücksetzen beim Wechsel
  };
  toggleWaves.onclick  =()=>{
    wavesEnabled = !wavesEnabled;
    toggleWaves.textContent = wavesEnabled ? 'Wellen: Ein' : 'Wellen: Aus';
  };

  // Gegnerwellen
  function spawnWave(){
    const conf = DIFFS[difficulty];
    const n = Math.max(0, Math.floor(conf.waveSize(res.wave)));
    if(n<=0) return;
    res.wave++;
    for(let i=0;i<n;i++){
      const edge = Math.floor(r(0,4));
      let x,y;
      if(edge===0){ x=-20;    y=r(60,H-60); }
      if(edge===1){ x=W+20;   y=r(60,H-60); }
      if(edge===2){ x=r(60,W-60); y=-20; }
      if(edge===3){ x=r(60,W-60); y=H+20; }
      enemy.push( raider(x,y) );
    }
    updateHUD();
  }

  // === Loop ==================================================================
  let last=0;
  function loop(ts){
    const dt=Math.min(0.033,(ts-last)/1000); last=ts;
    if(!paused){
      // Wellen-Timer
      if(wavesEnabled){
        const conf = DIFFS[difficulty];
        waveTimer += dt;
        const due = (res.wave === 0) ? conf.first : conf.every;
        if (waveTimer >= due) { waveTimer = 0; spawnWave(); }
      }

      stepUnits(friendly, dt, true);
      stepUnits(enemy, dt, false);
      farmsPassive(dt);
      cleanDead();
    }
    draw();
    requestAnimationFrame(loop);
  }

  // Drop-off Helfer
  function nearestDropoff(obj){
    let best=null, bd=1e9;
    for(const b of buildings){ if(b.dropoff){ const d=Math.hypot(b.x-obj.x,b.y-obj.y); if(d<bd){ bd=d; best=b; } } }
    return best;
  }

  function stepUnits(list, dt, isFriendly){
    for(const u of list){
      // Auto-Ziel für Gegner
      if(!isFriendly && (!u.target || u.target.hp<=0)){
        u.target = findNearestTarget(u, [...buildings, ...friendly]);
        if(u.target){ u.tx=u.target.x; u.ty=u.target.y; }
      }
      if(isFriendly && (u.task==='attack') && u.target && u.target.hp<=0){ u.task=null; }

      // Bewegung
      const dx=u.tx-u.x, dy=u.ty-u.y, d=Math.hypot(dx,dy);
      if(d>1){ const s=u.speed*dt; u.x += (dx/d)*Math.min(s,d); u.y += (dy/d)*Math.min(s,d); }

      // Sammeln (Villager, mit Tragekapazität + Drop-off)
      if(u.type==='villager'){
        if(u.task && d<=14){
          u._t += dt;
          if(u._t>=0.5){
            u._t=0;
            if(u.task==='wood' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='wood'; u.carry.amt+=2; }
            else if(u.task==='stone' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='stone'; u.carry.amt+=2; }
            else if(u.task==='gold' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='gold'; u.carry.amt+=2; }
            else if(u.task==='food-node' && u.target?.hp>0){ u.target.hp-=2; u.carry.type='food'; u.carry.amt+=2; }
            else if(u.task==='farm'){ u.carry.type='food'; u.carry.amt+=3; }

            // Kapazität erreicht → zum Drop-off
            if(u.carry.amt >= u.carry.cap){
              const dpo = nearestDropoff(u);
              if(dpo){ u.task='return'; u.target=dpo; u.tx=dpo.x; u.ty=dpo.y; }
            }
          }
        }
        // Abgabe
        if(u.task==='return' && d<=18){
          if(u.carry.type){ res[u.carry.type] += u.carry.amt; u.carry.amt=0; updateHUD(); }
          // zurück zur gemerkten Arbeit
          if(u.remember && u.remember.target?.hp>0){ setTask(u,u.remember.kind,u.remember.target); }
          else { u.task=null; }
        }
      }

      // Kampf
      const foes = isFriendly ? enemy : friendly;
      const tgt = (u.task==='attack' && u.target) ? u.target : nearestInSight(u, foes, u.type==='archer'? 110:150);
      if(tgt){
        if(dist(u,tgt)<=u.range+2){
          u.last += dt;
          if(u.last >= u.rate){ u.last = 0; damage(tgt, u.atk); }
        }else{
          // hinlaufen bis in Reichweite
          const ang = Math.atan2(tgt.y-u.y, tgt.x-u.x);
          const keep = u.type==='archer' ? u.range-6 : u.range-2;
          u.tx = tgt.x - Math.cos(ang)*keep;
          u.ty = tgt.y - Math.sin(ang)*keep;
        }
      }
    }
  }

  function nearestInSight(u, list, sight){
    let best=null, bd=1e9;
    for(const t of list){ if(t.hp>0){ const d=dist(u,t); if(d<bd && d<=sight){ bd=d; best=t; } } }
    return best;
  }
  function findNearestTarget(u, list){
    let best=null, bd=1e9;
    for(const t of list){ if(t.hp>0){ const d=dist(u,t); if(d<bd){ bd=d; best=t; } } }
    return best;
  }
  function damage(obj,amt){ obj.hp = clamp(obj.hp-amt, 0, obj.max); }

  function farmsPassive(dt){
    for(const b of buildings){
      if(b.type==='farm'){
        b._passive=(b._passive||0)+dt;
        if(b._passive>2){ b._passive=0; res.food+=1; updateHUD(); }
      }
    }
  }
  function cleanDead(){
    for(let i=friendly.length-1;i>=0;i--) if(friendly[i].hp<=0){ friendly.splice(i,1); updateHUD(); }
    for(let i=enemy.length-1;i>=0;i--) if(enemy[i].hp<=0){ enemy.splice(i,1); }
    for(let i=buildings.length-1;i>=0;i--) if(buildings[i].hp<=0){ buildings.splice(i,1); }
  }

  // === Render ================================================================
  function draw(){
    // Wiese + Grid
    ctx.fillStyle='#16321c'; ctx.fillRect(0,0,W,H);
    ctx.globalAlpha=.06; ctx.fillStyle='#fff';
    for(let y=0;y<H;y+=TILE) for(let x=0;x<W;x+=TILE) ctx.fillRect(x,y,1,1);
    ctx.globalAlpha=1;

    // Auswahl-Rechteck
    if(dragging && dragStart && _mouse){
      ctx.fillStyle='rgba(59,130,246,.15)'; ctx.strokeStyle='rgba(59,130,246,.8)'; ctx.setLineDash([6,4]);
      const x=dragStart.x, y=dragStart.y, w=_mouse.x-x, h=_mouse.y-y;
      ctx.fillRect(x,y,w,h); ctx.strokeRect(x,y,w,h); ctx.setLineDash([]);
    }

    // Ressourcen
    const blob=(x,y,rad,fill,stroke)=>{ctx.beginPath();ctx.arc(x,y,rad,0,Math.PI*2);ctx.fillStyle=fill;ctx.fill();ctx.lineWidth=2;ctx.strokeStyle=stroke;ctx.stroke();};
    const bar=(x,y,w,h,p,c)=>{ctx.fillStyle='#0009';ctx.fillRect(x-w/2,y,w,h);ctx.fillStyle=c;ctx.fillRect(x-w/2,y,w*p,h);ctx.strokeStyle='#000';ctx.strokeRect(x-w/2,y,w,h);}
    for(const t of trees){ if(t.hp>0){ blob(t.x,t.y,18,'#2ecc71','#14532d'); ctx.fillStyle='#8e5a2c'; ctx.fillRect(t.x-2,t.y+12,4,10);} }
    for(const s of rocks){ if(s.hp>0){ blob(s.x,s.y,18,'#94a3b8','#334155'); } }
    for(const g of golds){ if(g.hp>0){ blob(g.x,g.y,18,'#fbbf24','#92400e'); } }
    for(const b of berries){ if(b.hp>0){ blob(b.x,b.y,16,'#b91c1c','#7f1d1d'); } }

    // Gebäude
    for(const b of buildings){
      if(b.type==='house'){
        ctx.fillStyle='#9ca3af'; ctx.fillRect(b.x-18,b.y-18,36,36);
        ctx.fillStyle='#b91c1c'; ctx.beginPath(); ctx.moveTo(b.x-22,b.y-18); ctx.lineTo(b.x+22,b.y-18); ctx.lineTo(b.x,b.y-36); ctx.closePath(); ctx.fill();
        ctx.fillStyle='#1f2937'; ctx.fillRect(b.x-6,b.y+2,12,14);
      }else if(b.type==='farm'){
        ctx.fillStyle='#166534'; ctx.fillRect(b.x-20,b.y-20,40,40);
        ctx.fillStyle='rgba(255,255,255,.15)'; for(let i=-16;i<=16;i+=8){ ctx.fillRect(b.x-20,b.y+i,40,2); }
        ctx.strokeStyle='#34d399'; ctx.strokeRect(b.x-20,b.y-20,40,40);
      }else if(b.type==='mill'){
        ctx.fillStyle='#8b5cf6'; ctx.fillRect(b.x-21,b.y-21,42,42);
        ctx.fillStyle='#e5e7eb';
        ctx.beginPath(); ctx.moveTo(b.x,b.y-24); ctx.lineTo(b.x+6,b.y-6); ctx.lineTo(b.x-6,b.y-6); ctx.closePath(); ctx.fill();
        ctx.beginPath(); ctx.moveTo(b.x+24,b.y); ctx.lineTo(b.x+6,b.y+6); ctx.lineTo(b.x+6,b.y-6); ctx.closePath(); ctx.fill();
        ctx.beginPath(); ctx.moveTo(b.x,b.y+24); ctx.lineTo(b.x-6,b.y+6); ctx.lineTo(b.x+6,b.y+6); ctx.closePath(); ctx.fill();
        ctx.beginPath(); ctx.moveTo(b.x-24,b.y); ctx.lineTo(b.x-6,b.y-6); ctx.lineTo(b.x-6,b.y+6); ctx.closePath(); ctx.fill();
      }else if(b.type==='barracks'){
        ctx.fillStyle='#374151'; ctx.fillRect(b.x-26,b.y-26,52,52);
        ctx.fillStyle='#9ca3af'; ctx.fillRect(b.x-20,b.y-12,40,24);
        ctx.fillStyle='#e5e7eb'; ctx.fillRect(b.x-6,b.y-6,12,12);
        ctx.strokeStyle='#f59e0b'; ctx.setLineDash([6,4]); ctx.strokeRect(b.x-26,b.y-26,52,52); ctx.setLineDash([]);
     
