+++
title = 'Sea Battle'
date = '2026-11-18T18:48:00Z'
draft = false
description = 'sea battle'
+++
```

<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width,initial-scale=1">
<title>SEA BATTLE '80</title>
<style>
*{box-sizing:border-box}
body{
 margin:0;background:#080e20;color:#e5e5c8;
 font-family:monospace;text-align:center;
}
header{padding:14px 8px 8px}
h1{margin:0;color:#f6d65a;font-size:clamp(25px,5vw,42px);
 letter-spacing:5px;text-shadow:3px 3px #a23b35}
header p{color:#9bb8c9;font-size:12px;letter-spacing:2px}
#gamebox{max-width:1050px;margin:auto;padding:8px}
canvas{
 display:block;width:100%;height:auto;
 border:4px solid #a4a17d;background:#123d65;
 image-rendering:pixelated;cursor:crosshair;
}
#controls{
 display:flex;justify-content:center;gap:8px;
 flex-wrap:wrap;padding:12px 0
}
button{
 background:#253e59;color:#f5df83;
 border:2px solid #d1bd77;padding:10px 15px;
 font-family:monospace;font-weight:bold;cursor:pointer
}
button:hover{background:#42648a}
#help{font-size:12px;color:#a9bac8;line-height:1.8;padding:5px}
#status{color:#f5d45c;min-height:20px;font-size:13px}
</style>
</head>
<body>
<header>
<h1>SEA BATTLE '80</h1>
<p>TACTICAL NAVAL WARFARE • ARCADE EDITION</p>
</header>
<div id="gamebox">
<canvas id="game" width="960" height="600"></canvas>
<div id="status">WELCOME ADMIRAL — SELECT YOUR FLEET</div>
<div id="controls">
<button onclick="newGame()">NEW GAME</button>
<button onclick="togglePause()">PAUSE / RESUME</button>
<button onclick="fire()">FIRE WEAPONS [SPACE]</button>
<button onclick="retreat()">RETREAT [R]</button>
</div>
<div id="help">
CLICK YOUR BLUE FLEET TO SELECT • CLICK OCEAN TO MOVE<br>
WASD / ARROW KEYS TO NAVIGATE • SPACE TO FIRE<br>
DESTROY ENEMY FLEETS AND CAPTURE THE RED HARBOR
</div>
</div>
<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");
const W=960,H=600,CELL=40;
const cols=24,rows=15;
let fleets=[],islands=[],selected=null;
let paused=false,gameOver=false,last=0;
let particles=[],shots=[],message="";
let wave=0,score=0;
let keys={};

const colors={
 blue:"#48b8ed",lightBlue:"#b4eaff",
 red:"#ed5a55",lightRed:"#ffb0a0",
 water:"#174b73",deep:"#103958",
 gold:"#f6d65a",white:"#eee9c9"
};

function fleet(id,team,x,y,ships,type){
 return {
  id,team,x,y,ships,type,
  target:null, cooldown:0, alive:true
 };
}

function newGame(){
 fleets=[];
 particles=[];
 shots=[];
 score=0;
 paused=false;
 gameOver=false;
 selected=null;
 wave=0;

 islands=[
  {x:250,y:155,r:48},
  {x:420,y:345,r:65},
  {x:620,y:150,r:42},
  {x:760,y:410,r:56},
  {x:145,y:445,r:38}
 ];

 // Player fleet
 fleets.push(fleet(1,"blue",105,125,[
  {name:"Battleship",hp:5,max:5},
  {name:"Destroyer",hp:3,max:3},
  {name:"Cruiser",hp:4,max:4}
 ],"STRIKE FLEET"));

 fleets.push(fleet(2,"blue",120,300,[
  {name:"Carrier",hp:6,max:6},
  {name:"Destroyer",hp:3,max:3},
  {name:"Submarine",hp:2,max:2}
 ],"CARRIER GROUP"));

 fleets.push(fleet(3,"blue",105,490,[
  {name:"Transport",hp:3,max:3},
  {name:"Cruiser",hp:4,max:4},
  {name:"Destroyer",hp:3,max:3}
 ],"LANDING FLEET"));

 // Enemy fleet
 fleets.push(fleet(4,"red",830,115,[
  {name:"Battleship",hp:5,max:5},
  {name:"Cruiser",hp:4,max:4},
  {name:"Destroyer",hp:3,max:3}
 ],"ENEMY FLEET"));

 fleets.push(fleet(5,"red",825,300,[
  {name:"Carrier",hp:6,max:6},
  {name:"Submarine",hp:2,max:2},
  {name:"Destroyer",hp:3,max:3}
 ],"ENEMY CARRIER"));

 fleets.push(fleet(6,"red",825,490,[
  {name:"Transport",hp:3,max:3},
  {name:"Cruiser",hp:4,max:4},
  {name:"Destroyer",hp:3,max:3}
 ],"ENEMY LANDING"));

 selected=fleets[0];
 say("YOUR FLEETS ARE READY. SELECT A FLEET AND ADVANCE!");
}

function say(t){
 message=t;
 document.getElementById("status").textContent=t;
}

function dist(a,b){
 return Math.hypot(a.x-b.x,a.y-b.y);
}

function aliveShips(f){
 return f.ships.filter(s=>s.hp>0);
}

function aliveFleet(f){
 return aliveShips(f).length>0;
}

function nearestEnemy(f){
 let enemies=fleets.filter(e=>e.team!==f.team&&aliveFleet(e));
 enemies.sort((a,b)=>dist(f,a)-dist(f,b));
 return enemies[0];
}

function blocked(x,y){
 return islands.some(i=>Math.hypot(x-i.x,y-i.y)<i.r+20);
}

function moveFleet(f,dx,dy){
 let nx=Math.max(30,Math.min(W-30,f.x+dx));
 let ny=Math.max(65,Math.min(H-25,f.y+dy));
 if(!blocked(nx,ny)){
  f.x=nx;f.y=ny;
 }
}

function fire(){
 if(paused||gameOver||!selected)return;
 let enemy=nearestEnemy(selected);
 if(!enemy){
  say("NO ENEMY TARGETS");
  return;
 }
 if(dist(selected,enemy)>245){
  say("TARGET OUT OF RANGE — MOVE CLOSER");
  return;
 }
 let target=aliveShips(enemy);
 if(!target.length)return;

 let ship=target[Math.floor(Math.random()*target.length)];
 let damage=1+(Math.random()<.25?1:0);
 ship.hp-=damage;

 shots.push({
  x1:selected.x,y1:selected.y,
  x2:enemy.x,y2:enemy.y,
  time:.25,color:colors.gold
 });
 particles.push({
  x:enemy.x,y:enemy.y,
  time:.6
 });

 say(selected.type+" FIRES! "+ship.name+
  " TAKES "+damage+" DAMAGE!");

 if(!aliveFleet(enemy)){
  say("ENEMY FLEET DESTROYED! +100 POINTS");
  score+=100;
  enemy.alive=false;
 }

 checkVictory();
}

function retreat(){
 if(!selected||gameOver)return;
 selected.target={x:selected.team==="blue"?80:880,y:300};
 say(selected.type+" RETREATING TO HOME PORT");
}

function checkVictory(){
 let blue=fleets.filter(f=>f.team==="blue"&&aliveFleet(f));
 let red=fleets.filter(f=>f.team==="red"&&aliveFleet(f));

 if(!red.length){
  gameOver=true;
  say("VICTORY! THE ENEMY FLEET HAS BEEN DESTROYED!");
 }
 if(!blue.length){
  gameOver=true;
  say("DEFEAT! YOUR FLEET HAS BEEN DESTROYED!");
 }
}

function ai(dt){
 fleets.filter(f=>f.team==="red"&&aliveFleet(f)).forEach(f=>{
  let target=nearestEnemy(f);
  if(!target)return;

  if(dist(f,target)>180){
   let dx=target.x-f.x,dy=target.y-f.y;
   let d=Math.hypot(dx,dy)||1;
   moveFleet(f,dx/d*dt*25,dy/d*dt*25);
  }else{
   f.cooldown-=dt;
   if(f.cooldown<=0){
    f.cooldown=2.5+Math.random()*2;
    let ships=aliveShips(target);
    if(ships.length){
     let s=ships[Math.floor(Math.random()*ships.length)];
     s.hp--;
     particles.push({x:target.x,y:target.y,time:.5});
     shots.push({
      x1:f.x,y1:f.y,x2:target.x,y2:target.y,
      time:.3,color:colors.red
     });
     say("ENEMY FIRE! YOUR "+s.name+" HAS BEEN HIT!");
     if(!aliveFleet(target)){
      target.alive=false;
      say("YOUR FLEET HAS BEEN DESTROYED!");
     }
     checkVictory();
    }
   }
  }
 });
}

function update(dt){
 if(paused||gameOver)return;

 fleets.forEach(f=>{
  if(!aliveFleet(f))return;
  if(f.target){
   let dx=f.target.x-f.x,dy=f.target.y-f.y;
   let d=Math.hypot(dx,dy);
   if(d<5)f.target=null;
   else moveFleet(f,dx/d*dt*55,dy/d*dt*55);
  }
 });

 if(selected){
  let speed=150*dt;
  if(keys.ArrowLeft||keys.a)moveFleet(selected,-speed,0);
  if(keys.ArrowRight||keys.d)moveFleet(selected,speed,0);
  if(keys.ArrowUp||keys.w)moveFleet(selected,0,-speed);
  if(keys.ArrowDown||keys.s)moveFleet(selected,0,speed);
 }
 ai(dt);

 shots.forEach(s=>s.time-=dt);
 shots=shots.filter(s=>s.time>0);
 particles.forEach(p=>p.time-=dt);
 particles=particles.filter(p=>p.time>0);
}

function drawOcean(){
 ctx.fillStyle=colors.water;
 ctx.fillRect(0,0,W,H);

 for(let y=0;y<H;y+=40){
  for(let x=0;x<W;x+=40){
   ctx.fillStyle=((x/40+y/40)%2===0)?
    "#17466b":"#194d73";
   ctx.fillRect(x,y,40,40);
   ctx.strokeStyle="#276087";
   ctx.globalAlpha=.35;
   ctx.beginPath();
   ctx.moveTo(x+8,y+18);
   ctx.lineTo(x+23,y+18);
   ctx.stroke();
   ctx.globalAlpha=1;
  }
 }
}

function drawIsland(i){
 ctx.fillStyle="#b6a36c";
 ctx.beginPath();
 ctx.arc(i.x,i.y,i.r,0,Math.PI*2);
 ctx.fill();
 ctx.fillStyle="#587d48";
 ctx.beginPath();
 ctx.arc(i.x-4,i.y-5,i.r*.78,0,Math.PI*2);
 ctx.fill();
 ctx.fillStyle="#819e5c";
 ctx.fillRect(i.x-9,i.y-10,12,8);
 ctx.fillRect(i.x+10,i.y+6,8,9);
}

function drawHarbors(){
 // Blue home port
 ctx.fillStyle="#58bde6";
 ctx.fillRect(0,265,45,70);
 ctx.fillStyle="#d6e8d8";
 ctx.font="bold 12px monospace";
 ctx.fillText("BLUE",4,258);
 ctx.fillText("PORT",4,350);

 // Enemy harbor
 ctx.fillStyle="#ed5a55";
 ctx.fillRect(W-45,265,45,70);
 ctx.fillStyle="#fff0d0";
 ctx.fillText("RED",W-43,258);
 ctx.fillText("PORT",W-43,350);
}

function drawShip(f,index){
 let x=f.x+(index-1)*10;
 let y=f.y+(index-1)*13;
 let color=f.team==="blue"?colors.blue:colors.red;

 ctx.save();
 ctx.translate(x,y);
 ctx.fillStyle="#071b30";
 ctx.beginPath();
 ctx.moveTo(-15,0);
 ctx.lineTo(-8,-7);
 ctx.lineTo(13,-5);
 ctx.lineTo(18,0);
 ctx.lineTo(12,7);
 ctx.lineTo(-10,6);
 ctx.closePath();
 ctx.fill();

 ctx.fillStyle=color;
 ctx.beginPath();
 ctx.moveTo(-15,-2);
 ctx.lineTo(-8,-5);
 ctx.lineTo(12,-3);
 ctx.lineTo(16,0);
 ctx.lineTo(10,4);
 ctx.lineTo(-10,4);
 ctx.closePath();
 ctx.fill();

 ctx.fillStyle="#f0e8bd";
 ctx.fillRect(-4,-3,7,4);
 ctx.fillRect(4,-2,3,3);
 ctx.restore();
}

function drawFleet(f){
 if(!aliveFleet(f))return;
 let color=f.team==="blue"?colors.blue:colors.red;

 if(selected===f){
  ctx.strokeStyle=colors.gold;
  ctx.lineWidth=2;
  ctx.beginPath();
  ctx.arc(f.x,f.y,30,0,Math.PI*2);
  ctx.stroke();
 }
 if(f.target){
  ctx.strokeStyle=color;
  ctx.setLineDash([5,5]);
  ctx.beginPath();
  ctx.moveTo(f.x,f.y);
  ctx.lineTo(f.target.x,f.target.y);
  ctx.stroke();
  ctx.setLineDash([]);
 }
 aliveShips(f).forEach((s,i)=>drawShip(f,i));

 ctx.fillStyle="#fff2c5";
 ctx.font="10px monospace";
 ctx.textAlign="center";
 ctx.fillText(f.type,f.x,f.y+35);
 ctx.fillStyle=color;
 ctx.fillRect(f.x-24,f.y+40,48,4);

 let hp=aliveShips(f).reduce((a,s)=>a+s.hp,0);
 let max=f.ships.reduce((a,s)=>a+s.max,0);
 ctx.fillStyle="#66dd99";
 ctx.fillRect(f.x-24,f.y+40,48*hp/max,4);
 ctx.textAlign="left";
}

function draw(){
 drawOcean();
 islands.forEach(drawIsland);
 drawHarbors();

 // Firing effects
 shots.forEach(s=>{
  ctx.strokeStyle=s.color;
  ctx.lineWidth=3;
  ctx.globalAlpha=Math.min(1,s.time*4);
  ctx.beginPath();
  ctx.moveTo(s.x1,s.y1);
  ctx.lineTo(s.x2,s.y2);
  ctx.stroke();
  ctx.globalAlpha=1;
 });

 particles.forEach(p=>{
  ctx.fillStyle=colors.gold;
  ctx.globalAlpha=p.time*1.5;
  ctx.beginPath();
  ctx.arc(p.x,p.y,18*(1-p.time),0,Math.PI*2);
  ctx.fill();
  ctx.globalAlpha=1;
 });

 fleets.forEach(drawFleet);

 // Top HUD
 ctx.fillStyle="#061a2b";
 ctx.fillRect(0,0,W,43);
 ctx.fillStyle=colors.gold;
 ctx.font="bold 16px monospace";
 ctx.fillText("SEA BATTLE '80",16,27);
 ctx.fillStyle="#d6e4dd";
 ctx.font="12px monospace";
 ctx.fillText("SCORE: "+score,250,26);
 ctx.fillText("FLEETS: "+
  fleets.filter(f=>f.team==="blue"&&aliveFleet(f)).length+
  " BLUE / "+
  fleets.filter(f=>f.team==="red"&&aliveFleet(f)).length+
  " RED",410,26);

 if(paused||gameOver){
  ctx.fillStyle="rgba(0,0,0,.7)";
  ctx.fillRect(0,0,W,H);
  ctx.textAlign="center";
  ctx.fillStyle=colors.gold;
  ctx.font="bold 36px monospace";
  ctx.fillText(gameOver?"BATTLE COMPLETE":"GAME PAUSED",W/2,H/2);
  ctx.font="16px monospace";
  ctx.fillStyle="#fff";
  ctx.fillText(gameOver?"PRESS NEW GAME TO PLAY AGAIN":"PRESS PAUSE TO CONTINUE",W/2,H/2+35);
  ctx.textAlign="left";
 }
}

function loop(t){
 let dt=Math.min((t-last)/1000,.05)||0;
 last=t;
 update(dt);
 draw();
 requestAnimationFrame(loop);
}

canvas.addEventListener("click",e=>{
 if(paused||gameOver)return;
 let r=canvas.getBoundingClientRect();
 let x=(e.clientX-r.left)*W/r.width;
 let y=(e.clientY-r.top)*H/r.height;

 let clicked=fleets.find(f=>
  f.team==="blue"&&aliveFleet(f)&&dist({x,y},f)<38
 );

 if(clicked){
  selected=clicked;
  say("SELECTED: "+clicked.type+
   " — "+aliveShips(clicked).length+" SHIPS REMAINING");
 }else if(selected){
  selected.target={x,y};
  say("FLEET MOVING TO NEW POSITION");
 }
});

window.addEventListener("keydown",e=>{
 if(["ArrowUp","ArrowDown","ArrowLeft","ArrowRight"," "].includes(e.key))
  e.preventDefault();

 keys[e.key]=true;
 if(e.key===" ")fire();
 if(e.key.toLowerCase()==="r")retreat();
 if(e.key==="Escape")togglePause();
});

window.addEventListener("keyup",e=>keys[e.key]=false);

function togglePause(){
 if(!gameOver)paused=!paused;
}

newGame();
requestAnimationFrame(loop);
</script>
</body>
</html>

```

