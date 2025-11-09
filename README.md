# v-roba_mouky
vyrábíš mouku.
[hra_vyroba_mouky (1).html](https://github.com/user-attachments/files/23440108/hra_vyroba_mouky.1.html)
<!doctype html>
<html lang="cs">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Hra: mouka men</title>
  <style>
    :root{--bg:#f7f3ea;--panel:#fff;--accent:#8b5e3c}
    body{margin:0;font-family:Inter,system-ui,Segoe UI,Roboto,Arial; background:var(--bg); color:#222}
    header{padding:12px 18px;background:linear-gradient(90deg,#fff,#f2efe9);border-bottom:1px solid #e6dfd0;display:flex;align-items:center;gap:12px}
    h1{font-size:18px;margin:0}
    main{display:flex;gap:18px;padding:18px}
    #game{flex:1;background:var(--panel);border-radius:12px;box-shadow:0 6px 18px rgba(0,0,0,.06);padding:14px}
    #sidebar{width:300px;background:var(--panel);border-radius:12px;padding:14px;box-shadow:0 6px 18px rgba(0,0,0,.04)}
    canvas{background:linear-gradient(#fbf8f2,#fff);display:block;border-radius:10px;width:100%;height:520px}
    .controls{display:flex;gap:8px;margin-top:8px}
    button{background:var(--accent);color:#fff;border:0;padding:8px 10px;border-radius:8px;cursor:pointer}
    button:disabled{opacity:.5;cursor:default}
    .stat{margin-bottom:10px}
    .small{font-size:13px;color:#555}
    .meter{height:12px;background:#eee;border-radius:8px;overflow:hidden}
    .meter > i{display:block;height:100%;background:linear-gradient(#ffdca8,#f2b97f)}
    footer{padding:12px;text-align:center;color:#666;font-size:13px}
  </style>
</head>
<body>
  <header>
    <h1>Hra: Výroba mouky — Mlýn</h1>
    <div class="small">Cíl: projdi každou fázi od zrna po mouku a získej co nejvyšší kvalitu!</div>
  </header>
  <main>
    <section id="game">
      <canvas id="c" width="900" height="520"></canvas>
      <div class="controls">
        <button id="btnStart">Nová hra</button>
        <button id="btnRestart" disabled>Restart</button>
        <button id="btnHint">Tip</button>
      </div>
    </section>
    <aside id="sidebar">
      <div class="stat"><strong>Fáze:</strong> <span id="phase">—</span></div>
      <div class="stat"><strong>Skóre:</strong> <span id="score">0</span></div>
      <div class="stat"><strong>Kvalita mouky:</strong>
        <div class="meter" style="margin-top:6px"><i id="quality" style="width:0%"></i></div>
      </div>
      <hr />
      <div class="stat"><strong>Instrukce</strong>
        <div class="small" id="instructions">Stiskni "Nová hra" pro začátek.</div>
      </div>
      <hr />
      <div class="stat"><strong>Tipy</strong>
        <ol class="small">
          <li>Správné čištění zvyšuje kvalitu.</li>
          <li>Příliš mokré zrno zhorší mletí.</li>
          <li>Vyrovnej parametry mletí pro hladkou mouku.</li>
        </ol>
      </div>
    </aside>
  </main>
  <footer>Vyrobil: ChatGPT — jednoduchá prohlížečová hra. Chceš úpravy? Napiš jak!</footer>

<script>
// Jednosouborová hra o výrobě mouky
(() => {
  const canvas = document.getElementById('c');
  const ctx = canvas.getContext('2d');
  const W = canvas.width, H = canvas.height;
  const phaseEl = document.getElementById('phase');
  const scoreEl = document.getElementById('score');
  const qualityEl = document.getElementById('quality');
  const instr = document.getElementById('instructions');
  const btnStart = document.getElementById('btnStart');
  const btnRestart = document.getElementById('btnRestart');
  const btnHint = document.getElementById('btnHint');

  let game = null;

  const PHASES = ['sklizen', 'cisteni', 'kondicionovani', 'mleti', 'baleni', 'hotovo'];

  function newGame(){
    game = {
      phaseIndex: 0,
      score: 0,
      quality: 50, // 0-100
      grain: [],
      time: 0,
      tick: 0,
      target: 0,
      millingPower: 50,
      moisture: 50,
      alive:true
    };
    // generate grains
    for(let i=0;i<8;i++) game.grain.push({x:60+i*60,y:80+Math.random()*40, vx:0,selected:false});
    updateUI();
    instr.innerText = 'Přetáhni zrna do násypky (pod tímto prknem) myší. Pak klikni "Čistit".';
    loop();
  }

  function updateUI(){
    phaseEl.innerText = PHASES[game.phaseIndex];
    scoreEl.innerText = game.score;
    qualityEl.style.width = game.quality + '%';
  }

  // Simple mouse interactions
  let mouse = {x:0,y:0,down:false,drag:null};
  canvas.addEventListener('mousemove', e=>{
    const rect = canvas.getBoundingClientRect();
    mouse.x = (e.clientX-rect.left)*(canvas.width/rect.width);
    mouse.y = (e.clientY-rect.top)*(canvas.height/rect.height);
  });
  canvas.addEventListener('mousedown', e=>{ mouse.down=true; pickGrain(); });
  canvas.addEventListener('mouseup', e=>{ mouse.down=false; releaseGrain(); });

  function pickGrain(){
    if(!game) return;
    for(const g of game.grain){
      if(dist(mouse.x,mouse.y,g.x,g.y) < 18){ g.selected = true; game.tick++; mouse.drag=g; break; }
    }
  }
  function releaseGrain(){
    if(mouse.drag) mouse.drag.selected=false;
    mouse.drag=null;
    // check if dropped in hopper
    if(game.phaseIndex===0){
      // hopper area x:350-560 y:260-420
      for(const g of game.grain){
        if(g.x>350 && g.x<560 && g.y>260 && g.y<420){
          // move to hopper
          g.inHopper = true;
          game.score += 5;
        }
      }
    }
  }

  function dist(x1,y1,x2,y2){ return Math.hypot(x1-x2,y1-y2); }

  // Buttons
  btnStart.addEventListener('click', ()=>{
    newGame();
    btnRestart.disabled=false;
  });
  btnRestart.addEventListener('click', ()=>{ if(game) newGame(); });
  btnHint.addEventListener('click', ()=>{ if(!game) return; giveHint(); });

  function giveHint(){
    const ph = PHASES[game.phaseIndex];
    let t='';
    if(ph==='sklizen') t='Dostaň aspoň 6 zrnek do násypky.';
    else if(ph==='cisteni') t='Klikni ve správný moment, když rámec osciluje uprostřed.';
    else if(ph==='kondicionovani') t='Uprav vlhkost na ~40–55%. Příliš mokré=horší mletí.';
    else if(ph==='mleti') t='Zvyš sílu mletí postupně a drž stabilní otáčky.';
    else if(ph==='baleni') t='Klikni rychle a pravidelně pro naplnění pytlů.';
    instr.innerText = t;
  }

  // Game loop and drawing
  function loop(){
    if(!game) return;
    game.time += 1/60;
    game.tick++;
    update();
    draw();
    if(game.alive) requestAnimationFrame(loop);
  }

  function update(){
    // dragging
    if(mouse.drag){ mouse.drag.x = mouse.x; mouse.drag.y = mouse.y; }

    const ph = PHASES[game.phaseIndex];
    if(ph==='sklizen'){
      // check if enough in hopper
      const inHopper = game.grain.filter(g=>g.inHopper).length;
      if(inHopper>=6){
        game.phaseIndex++;
        instr.innerText = 'Čištění: klikni na "Čistit" (v prostoru vedle) ve správný moment.';
      }
    } else if(ph==='cisteni'){
      // we simulate an oscillating bar the player must stop within range
      // automatic progression handled on button press
    } else if(ph==='kondicionovani'){
      // player will adjust moisture slider via keys
    } else if(ph==='mleti'){
      // milling simulation
    } else if(ph==='baleni'){
      // packaging simulation
    }
    updateUI();
  }

  // Drawing UI and game world
  function draw(){
    ctx.clearRect(0,0,W,H);
    drawBackground();
    drawGrain();
    drawHopper();
    drawMill();
    drawUIOverlay();
  }

  function drawBackground(){
    // sky
    const g = ctx.createLinearGradient(0,0,0,H);
    g.addColorStop(0,'#fff'); g.addColorStop(1,'#f3efe6');
    ctx.fillStyle = g; ctx.fillRect(0,0,W,H);
  }

  function drawGrain(){
    for(const g of game.grain){
      ctx.beginPath();
      ctx.ellipse(g.x,g.y,12,8,0,0,Math.PI*2);
      ctx.fillStyle = g.inHopper? '#c9a15a' : '#e7c67a';
      ctx.fill();
      ctx.strokeStyle = '#b38a4a'; ctx.stroke();
    }
  }

  function drawHopper(){
    // draw a simple hopper at center-right
    ctx.fillStyle='#f0e6d6'; ctx.strokeStyle='#d4c7b0'; ctx.lineWidth=2;
    ctx.beginPath(); ctx.moveTo(320,230); ctx.lineTo(580,230); ctx.lineTo(520,420); ctx.lineTo(380,420); ctx.closePath();
    ctx.fill(); ctx.stroke();
    ctx.fillStyle='#89623f'; ctx.fillRect(360,200,120,20);
    ctx.fillStyle='#333'; ctx.font='16px sans-serif'; ctx.fillText('Násypka', 380, 195);

    // show count
    const cnt = game.grain.filter(g=>g.inHopper).length;
    ctx.fillStyle='#333'; ctx.font='20px sans-serif'; ctx.fillText(cnt + ' / 8', 460, 400);
  }

  function drawMill(){
    // millstones on left
    ctx.save(); ctx.translate(130,350);
    // base
    ctx.fillStyle='#efe7db'; ctx.fillRect(-90,-80,180,160);
    // stone
    ctx.beginPath(); ctx.arc(0,0,60,0,Math.PI*2); ctx.fillStyle='#bfbfbf'; ctx.fill(); ctx.strokeStyle='#9e9e9e'; ctx.stroke();
    // spindle
    ctx.fillStyle='#8b5e3c'; ctx.fillRect(-6,-60,12,40);
    ctx.restore();
  }

  function drawUIOverlay(){
    // draw current phase box
    ctx.fillStyle='rgba(255,255,255,0.9)'; ctx.fillRect(10,10,240,68);
    ctx.strokeStyle='#e5dfd2'; ctx.strokeRect(10,10,240,68);
    ctx.fillStyle='#333'; ctx.font='14px sans-serif'; ctx.fillText('Fáze: ' + PHASES[game.phaseIndex], 22, 34);
    ctx.fillStyle='#666'; ctx.font='12px sans-serif'; ctx.fillText('Skóre: ' + game.score, 22, 54);

    // context sensitive controls
    const ph = PHASES[game.phaseIndex];
    ctx.fillStyle='#222'; ctx.font='13px sans-serif';
    if(ph==='sklizen'){
      ctx.fillText('Přetáhni zrna do násypky (uprostřed).', 270, 34);
    } else if(ph==='cisteni'){
      // draw oscillating bar
      const t = (Date.now()/300) % (Math.PI*2);
      const x = 430 + Math.sin(t)*90;
      ctx.fillStyle='#fff'; ctx.fillRect(260,40,420,36); ctx.strokeRect(260,40,420,36);
      ctx.fillStyle='#cfcfcf'; ctx.fillRect(430,48,100,20);
      // sweet spot
      ctx.fillStyle='rgba(120,200,120,0.5)'; ctx.fillRect(420,44,40,28);
      ctx.fillStyle='#333'; ctx.fillText('Čištění: klikni na "Čistit" když bílý pruh projde zelenou zónou.', 270, 66);
      // create button
      drawButton(700,58,'Čistit', ()=>{ doCleaning(); });
    } else if(ph==='kondicionovani'){
      // slider for moisture
      ctx.fillText('Kondicionování: nastav vlhkost (A/D) a potvrď SPACE.', 270, 34);
      ctx.fillText('Vlhost: ' + game.moisture + '%', 270, 58);
    } else if(ph==='mleti'){
      ctx.fillText('Mletí: zvol sílu (W/S) a drž stabilní výkon 3s pro dobrý výsledek.', 270, 34);
      ctx.fillText('Síla: ' + game.millingPower + '%', 270, 58);
    } else if(ph==='baleni'){
      ctx.fillText('Balení: klikni rychle pro naplnění pytlů (klikni v canvasu).', 270, 34);
    } else if(ph==='hotovo'){
      ctx.fillText('Hotovo — těleso: kvalitní mouka!', 270, 34);
    }
  }

  // draw fake button (visual only) — still call provided handler on real button press
  function drawButton(x,y,label,handler){
    ctx.fillStyle='#8b5e3c'; ctx.fillRect(x-8,y-18,70,28);
    ctx.fillStyle='#fff'; ctx.font='13px sans-serif'; ctx.fillText(label, x+6, y);
  }

  // Game actions
  window.addEventListener('keydown', e=>{
    if(!game) return;
    if(PHASES[game.phaseIndex]==='kondicionovani'){
      if(e.key.toLowerCase()==='a'){ game.moisture = Math.max(10, game.moisture-3); }
      if(e.key.toLowerCase()==='d'){ game.moisture = Math.min(90, game.moisture+3); }
      if(e.code==='Space'){ confirmConditioning(); }
    }
    if(PHASES[game.phaseIndex]==='mleti'){
      if(e.key.toLowerCase()==='w'){ game.millingPower = Math.min(100, game.millingPower+3); }
      if(e.key.toLowerCase()==='s'){ game.millingPower = Math.max(10, game.millingPower-3); }
      if(e.code==='Space'){ confirmMilling(); }
    }
  });

  // Cleaning simulated by checking timing when clicked on HUD 'Čistit' region
  document.addEventListener('click', e=>{
    if(!game) return;
    if(PHASES[game.phaseIndex]==='cisteni'){
      // approximate if click was within the green spot time
      // compute sin value
      const t = (Date.now()/300) % (Math.PI*2);
      const val = Math.sin(t);
      if(val>0.7){ // good
        game.quality = Math.min(100, game.quality + 12);
        game.score += 15;
        instr.innerText = 'Skvěle! Otruby odstraněny.';
      } else if(val>0.2){
        game.quality = Math.min(100, game.quality + 6);
        game.score += 8;
        instr.innerText = 'Dobře — částečně vyčištěno.';
      } else {
        game.quality = Math.max(0, game.quality - 8);
        game.score -= 5;
        instr.innerText = 'Špatný moment — zůstaly nečistoty.';
      }
      // progress
      game.phaseIndex++; updateUI();
    }

    if(PHASES[game.phaseIndex]==='baleni'){
      // clicking in canvas counts as filling
      game.score += 2;
      game.quality = Math.min(100, game.quality + Math.random()*2);
      if(game.score > 120){ finishGame(); }
    }
  });

  function doCleaning(){
    // simulate pressing cleaning button — we treat it same as clicking anywhere
    if(game && PHASES[game.phaseIndex]==='cisteni'){
      // emulate click
      const evt = new MouseEvent('click'); document.dispatchEvent(evt);
    }
  }

  function confirmConditioning(){
    // moisture affects quality: ideal ~45
    const ideal = 45; const diff = Math.abs(game.moisture - ideal);
    if(diff <= 7){ game.quality = Math.min(100, game.quality + 10); game.score += 12; instr.innerText='Vlhkost ideální!'; }
    else if(diff <= 20){ game.quality = Math.min(100, game.quality + 4); game.score += 6; instr.innerText='Vlhkost přijatelná.'; }
    else { game.quality = Math.max(0, game.quality - 10); game.score -= 6; instr.innerText='Příliš extrémní vlhkost, kvalita klesá.'; }
    game.phaseIndex++; updateUI();
  }

  function confirmMilling(){
    // millingPower must be near 60 and moisture near 45 for best
    const powerDiff = Math.abs(game.millingPower - 60);
    const moistDiff = Math.abs(game.moisture - 45);
    let gain = 0;
    if(powerDiff <= 8 && moistDiff <= 10) { gain = 20; game.score += 30; instr.innerText='Mletí perfektní!'; }
    else if(powerDiff <= 20) { gain = 8; game.score += 12; instr.innerText='Mletí OK.'; }
    else { gain = -12; game.score -= 10; instr.innerText='Mletí nevyhovující — mouka hrubá.'; }
    game.quality = Math.max(0, Math.min(100, game.quality + gain));
    game.phaseIndex++; updateUI();
  }

  function finishGame(){
    game.phaseIndex = PHASES.indexOf('hotovo');
    // final score adjusts by quality
    const bonus = Math.round(game.quality/2);
    game.score += bonus;
    instr.innerText = 'Hotovo! Kvalita mouky: ' + Math.round(game.quality) + ' — zisk bodů: +' + bonus;
    updateUI();
    game.alive=false;
  }

  // if player reaches packaging without finishing, allow clicking many times
  // small auto progress when enough score
  setInterval(()=>{
    if(!game) return;
    if(PHASES[game.phaseIndex]==='mleti'){
      // mild drift: if player doesn't confirm, auto-fail slowly
      if(game.tick%180===0){ game.quality = Math.max(0, game.quality-1); }
    }
    if(PHASES[game.phaseIndex]==='kondicionovani'){
      // hint in instructions
    }
  },1000);

  // minimal start screen draw
  (function intro(){
    ctx.fillStyle='#fff'; ctx.fillRect(0,0,W,H);
    ctx.fillStyle='#333'; ctx.font='28px sans-serif'; ctx.fillText('Výroba mouky — Klikni "Nová hra"', 180, 220);
  })();

})();
</script>
</body>
</html>
