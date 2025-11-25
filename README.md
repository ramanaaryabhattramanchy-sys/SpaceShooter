<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
<title>2D Shooter Game</title>
<style>
body {
    margin: 0;
    display: flex;
    justify-content: center;
    align-items: center;
    height: 100vh;
    background: black;
    font-family: Arial, sans-serif;
    color: white;
    overflow: hidden;
    touch-action: none;
}
#game-container {
    position: relative;
    width: 100%;
    max-width: 600px;
}
canvas {
    width: 100%;
    height: auto;
    display: block;
    background: black;
}
#ui {
    position: absolute;
    top: 10px;
    left: 10px;
    font-size: 4vw;
}
#health-bar {
    position: absolute;
    top: 40px;
    left: 10px;
    width: 100px;
    height: 10px;
    border: 2px solid white;
}
#health-inner {
    height: 100%;
    background: lime;
    width: 100%;
}
#game-over {
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
    font-size: 6vw;
    color: red;
    display: none;
    text-align: center;
}
#restart-btn {
    position: absolute;
    top: 60%;
    left: 50%;
    transform: translate(-50%, -50%);
    padding: 10px 20px;
    font-size: 5vw;
    background: white;
    color: black;
    border-radius: 10px;
    display: none;
    user-select: none;
}
#controls {
    position: absolute;
    bottom: 10px;
    width: 100%;
    display: flex;
    justify-content: space-around;
}
.btn {
    width: 20%;
    max-width: 100px;
    height: 50px;
    background: white;
    color: black;
    border-radius: 10px;
    font-size: 4vw;
    user-select: none;
}
</style>
</head>
<body>
<div id="game-container">
    <canvas id="gameCanvas" width="600" height="500"></canvas>
    <div id="ui">
        <div id="score">Score: 0</div>
        <div id="level">Level: 1</div>
    </div>
    <div id="health-bar"><div id="health-inner"></div></div>
    <div id="game-over">Game Over!</div>
    <button id="restart-btn">Restart</button>
    <div id="controls">
        <div class="btn" id="left-btn">⬅</div>
        <div class="btn" id="shoot-btn">Shoot</div>
        <div class="btn" id="right-btn">➡</div>
    </div>
</div>
<audio id="shoot-sound" src="https://freesound.org/data/previews/66/66717_931655-lq.mp3"></audio>
<audio id="explosion-sound" src="https://freesound.org/data/previews/174/174024_3242494-lq.mp3"></audio>

<script>
const canvas = document.getElementById('gameCanvas');
const ctx = canvas.getContext('2d');
const shootSound = document.getElementById('shoot-sound');
const explosionSound = document.getElementById('explosion-sound');

const scoreText = document.getElementById('score');
const levelText = document.getElementById('level');
const healthInner = document.getElementById('health-inner');
const gameOverText = document.getElementById('game-over');
const restartBtn = document.getElementById('restart-btn');

const player = {
    width:40,height:40,x:canvas.width/2-20,y:canvas.height-60,
    speed:5,bullets:[],canShoot:true,shield:false,doubleBullets:false,health:3
};

const keys = {};
let enemies=[], enemyRows=3, enemyCols=7, enemyWidth=40, enemyHeight=30, enemyPadding=20, enemyDirection=1, enemySpeed=1;
let enemyBullets=[], explosions=[];
let score=0, level=1, gameOver=false;

const stars=[];
for(let i=0;i<50;i++) stars.push({x:Math.random()*canvas.width,y:Math.random()*canvas.height,size:Math.random()*2});

function createEnemies(){
    enemies=[];
    for(let r=0;r<enemyRows;r++){
        for(let c=0;c<enemyCols;c++){
            enemies.push({x:c*(enemyWidth+enemyPadding)+50,y:r*(enemyHeight+enemyPadding)+30,width:enemyWidth,height:enemyHeight,alive:true});
        }
    }
}
createEnemies();

document.addEventListener('keydown', e=>keys[e.key]=true);
document.addEventListener('keyup', e=>keys[e.key]=false);
document.addEventListener('keydown', e=>{
    if((e.key==='r'||e.key==='R') && gameOver) restartGame();
    if(e.key==='r'||e.key==='R') shoot();
});

document.getElementById('left-btn').addEventListener('touchstart', ()=> keys['ArrowLeft']=true);
document.getElementById('left-btn').addEventListener('touchend', ()=> keys['ArrowLeft']=false);
document.getElementById('right-btn').addEventListener('touchstart', ()=> keys['ArrowRight']=true);
document.getElementById('right-btn').addEventListener('touchend', ()=> keys['ArrowRight']=false);
document.getElementById('shoot-btn').addEventListener('touchstart', shoot);
restartBtn.addEventListener('touchstart', restartGame);

function update(){
    if(gameOver) return;

    if(keys['ArrowLeft'] && player.x>0) player.x-=player.speed;
    if(keys['ArrowRight'] && player.x+player.width<canvas.width) player.x+=player.speed;

    player.bullets.forEach((b,i)=>{b.y-=b.speed;if(b.y<0) player.bullets.splice(i,1);});
    let changeDir=false;
    enemies.forEach(e=>{if(!e.alive) return; e.x+=enemyDirection*enemySpeed;if(e.x+e.width>=canvas.width||e.x<=0) changeDir=true;});
    if(changeDir){enemyDirection*=-1; enemies.forEach(e=>e.y+=20);}

    enemies.forEach(e=>{if(!e.alive) return; if(Math.random()<0.002*level) enemyBullets.push({x:e.x+e.width/2-2,y:e.y+e.height,width:4,height:10,speed:3+level});});

    enemyBullets.forEach((b,i)=>{
        b.y+=b.speed;
        if(b.y>canvas.height) enemyBullets.splice(i,1);
        if(!player.shield && b.x<player.x+player.width && b.x+b.width>player.x && b.y<player.y+player.height && b.y+b.height>player.y){
            enemyBullets.splice(i,1);
            player.health--;
            updateHealth();
            if(player.health<=0) gameOverScreen();
        }
    });

    player.bullets.forEach((b,bIndex)=>{
        enemies.forEach(e=>{
            if(!e.alive) return;
            if(b.x<e.x+e.width && b.x+b.width>e.x && b.y<e.y+e.height && b.y+b.height>e.y){
                e.alive=false; player.bullets.splice(bIndex,1); score+=10; scoreText.textContent="Score: "+score; explosionSound.play();
                explosions.push({x:e.x+e.width/2,y:e.y+e.height/2,radius:0,max:20});
            }
        });
    });

    explosions.forEach((ex,i)=>{
        ex.radius+=2;
        if(ex.radius>=ex.max) explosions.splice(i,1);
    });

    if(enemies.every(e=>!e.alive)) nextLevel();

    draw();
    requestAnimationFrame(update);
}

function shoot(){
    if(!player.canShoot) return;
    if(player.doubleBullets){
        player.bullets.push({x:player.x+5,y:player.y,width:4,height:10,speed:7});
        player.bullets.push({x:player.x+player.width-9,y:player.y,width:4,height:10,speed:7});
    }else player.bullets.push({x:player.x+player.width/2-2,y:player.y,width:4,height:10,speed:7});
    player.canShoot=false; shootSound.play(); setTimeout(()=>player.canShoot=true,300);
}

function nextLevel(){
    level++; enemySpeed+=0.5; levelText.textContent="Level: "+level; createEnemies(); enemyBullets=[];
    player.doubleBullets=Math.random()<0.3; player.shield=Math.random()<0.2;
    setTimeout(()=>{player.doubleBullets=false; player.shield=false;},10000);
}

function draw(){
    ctx.clearRect(0,0,canvas.width,canvas.height);
    ctx.fillStyle='white'; stars.forEach(s=>{ctx.fillRect(s.x,s.y,s.size,s.size); s.y+=0.5; if(s.y>canvas.height)s.y=0;});
    ctx.fillStyle=player.shield?'cyan':'lime'; ctx.fillRect(player.x,player.y,player.width,player.height);
    ctx.fillStyle='yellow'; player.bullets.forEach(b=>ctx.fillRect(b.x,b.y,b.width,b.height));
    ctx.fillStyle='red'; enemies.forEach(e=>{if(e.alive)ctx.fillRect(e.x,e.y,e.width,e.height);});
    ctx.fillStyle='orange'; enemyBullets.forEach(b=>ctx.fillRect(b.x,b.y,b.width,b.height));
    ctx.fillStyle='white'; explosions.forEach(ex=>{ctx.beginPath();ctx.arc(ex.x,ex.y,ex.radius,0,Math.PI*2);ctx.stroke();});
}

function updateHealth(){healthInner.style.width=(player.health/3*100)+'%';}

function gameOverScreen(){
    gameOver=true; gameOverText.style.display='block'; restartBtn.style.display='block';
}

function restartGame(){
    player.bullets=[]; enemyBullets=[]; explosions=[]; player.x=canvas.width/2-20;
    player.health=3; updateHealth();
    score=0; level=1; scoreText.textContent="Score: 0"; levelText.textContent="Level: 1";
    gameOver=false; gameOverText.style.display='none'; restartBtn.style.display='none'; enemySpeed=1;
    createEnemies(); update();
}

update();
</script>
</body>
</html>
