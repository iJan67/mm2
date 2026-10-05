# mm2
<!doctype html>
<html lang="en"><head><meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
<title>Mystery Mansion 3D</title>
<style>
:root{--bg:#e4e8f0;--fg:#1b2030;--card:#fff;--btn:#2f3f8f;--btnfg:#fff;box-sizing:border-box;padding-top:env(safe-area-inset-top,0px);padding-bottom:env(safe-area-inset-bottom,0px)}
@media (prefers-color-scheme:dark){:root:not([data-theme="light"]){--bg:#0e1020;--fg:#e8ebf5;--card:#1b1f38;--btn:#7c8cff;--btnfg:#0e1020}}
:root[data-theme="dark"]{--bg:#0e1020;--fg:#e8ebf5;--card:#1b1f38;--btn:#7c8cff;--btnfg:#0e1020}
html{height:100%;scroll-padding-top:env(safe-area-inset-top,0px)}
body{margin:0;height:100%;background:var(--bg);color:var(--fg);font:16px/1.4 "Trebuchet MS",system-ui,sans-serif}
#app{height:100%;box-sizing:border-box;display:flex;flex-direction:column;align-items:center;gap:8px;padding:8px}
.hud{width:100%;max-width:800px;display:flex;justify-content:space-between;font-weight:700}
.stage{position:relative;width:100%;max-width:800px}
canvas{display:block;width:100%;aspect-ratio:800/560;border-radius:10px;touch-action:none;background:#111}
.ov{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;background:rgba(10,12,25,.6);border-radius:10px}
.ov[hidden]{display:none}
.card{background:var(--card);padding:16px 18px;border-radius:12px;max-width:320px;text-align:center}
.card h2{margin:0 0 6px;font-size:21px}.card p{margin:0 0 12px;font-size:15px}
#maps{display:flex;gap:6px;justify-content:center;flex-wrap:wrap;margin-bottom:12px}
button{font:inherit;font-weight:700;border:0;border-radius:10px;padding:12px 20px;background:var(--btn);color:var(--btnfg);cursor:pointer}
button:disabled{opacity:.45}
.chip{padding:7px 10px;font-size:13px;background:transparent;color:var(--fg);border:2px solid var(--btn)}
.chip.sel{background:var(--btn);color:var(--btnfg)}
#act{width:100%;max-width:800px;padding:16px;font-size:18px}
#toast{position:absolute;top:8px;left:50%;transform:translateX(-50%);background:var(--card);padding:6px 12px;border-radius:20px;font-size:14px;opacity:0;transition:opacity .2s;pointer-events:none;white-space:nowrap}
#toast.on{opacity:1}
</style></head><body>
<div id="app">
<div class="hud"><span id="role">Role</span><span id="time">2:00</span><span id="alive">Alive 8</span></div>
<div class="stage"><canvas id="cv" width="800" height="560"></canvas><div id="toast"></div>
<div class="ov" id="ov"><div class="card"><h2 id="ot"></h2><p id="op"></p><div id="maps"></div><button id="ob">Start round</button></div></div></div>
<button id="act">Survive</button>
</div>
<script src="https://cdnjs.cloudflare.com/ajax/libs/three.js/r128/three.min.js"></script>
<script>
const W=800,H=560,R=12,STAB=30,SHOOT=260,ROUND=120,HX=400,HY=280;
const MAPS=[
{n:'🏚 Mansion',fl:'#23264a',fl2:'#2c3060',wl:0x6f78b0,h:30,pat:'check',bg:0x0e1020,
 w:[[190,110,18,170],[190,360,18,130],[592,70,18,150],[592,320,18,170],[310,240,180,18],[340,410,120,18],[80,250,70,18],[660,250,70,18],[390,90,18,90]]},
{n:'📦 Warehouse',fl:'#2b3336',fl2:'#e0b43a',wl:0xc27d2c,h:38,pat:'lanes',bg:0x121618,
 w:[[100,100,220,22],[480,100,220,22],[100,220,160,22],[340,220,120,22],[540,220,160,22],[100,340,220,22],[480,340,220,22],[250,450,300,22],[360,130,40,40],[420,360,40,40]]},
{n:'⛲ Courtyard',fl:'#2e5a3c',fl2:'#9aa38f',wl:0x4f8a4f,h:22,pat:'path',bg:0x0d1b14,
 w:[[200,150,28,28],[572,150,28,28],[200,382,28,28],[572,382,28,28],[360,240,80,80],[100,60,200,16],[500,60,200,16],[100,484,200,16],[500,484,200,16],[60,200,16,160],[724,200,16,160]]}];
const NAMES=['You','Alex','Bo','Cat','Dre','Eli','Fay','Gus'];
const INFO={m:['You are the Murderer 🔪','Tap a player to chase and stab them. Eliminate everyone before time runs out. Avoid the sheriff.'],
s:['You are the Sheriff 🔫','Find the murderer and tap them to shoot. Shoot an innocent and you die too.'],
i:['You are Innocent 🙂','Survive until the timer ends. If the sheriff falls, grab the dropped gun and stop the murderer.']};
const cv=document.getElementById('cv'),$=id=>document.getElementById(id);
const rd=new THREE.WebGLRenderer({canvas:cv,antialias:false,powerPreference:'high-performance'});rd.setPixelRatio(Math.min(devicePixelRatio,1.25));
const scene=new THREE.Scene(),cam=new THREE.PerspectiveCamera(45,W/H,1,3000);
cam.position.set(0,760,540);cam.lookAt(0,0,10);
scene.add(new THREE.AmbientLight(0xffffff,.7));const sun=new THREE.DirectionalLight(0xffffff,.65);sun.position.set(-250,500,300);scene.add(sun);
const gBody=new THREE.CylinderGeometry(9,10,20,14),gHead=new THREE.SphereGeometry(8,14,10),gVis=new THREE.BoxGeometry(9,4,4),gBlob=new THREE.CircleGeometry(13,16),gRingP=new THREE.RingGeometry(14,17,24),gFx=new THREE.RingGeometry(10,13,24);
const mSkin=new THREE.MeshLambertMaterial({color:0xf1c9a5}),mDark=new THREE.MeshLambertMaterial({color:0x1a1a22}),mBlob=new THREE.MeshBasicMaterial({color:0,transparent:true,opacity:.3}),mWhite=new THREE.MeshBasicMaterial({color:0xffffff});
let world,walls=[],ents=[],fx=[],gun=null,gunM=null,known=false,state='intro',timer=ROUND,player,keys={},tt,mapIdx=0;
const D=(a,b)=>Math.hypot(a.x-b.x,a.y-b.y);
function hitWall(x,y,r){if(x<r||y<r||x>W-r||y>H-r)return true;for(const[a,b,c,d]of walls){const nx=Math.max(a,Math.min(x,a+c)),ny=Math.max(b,Math.min(y,b+d));if((x-nx)**2+(y-ny)**2<r*r)return true}return false}
function los(a,b){const dx=b.x-a.x,dy=b.y-a.y,n=Math.ceil(Math.hypot(dx,dy)/6);for(let i=1;i<n;i++){const x=a.x+dx*i/n,y=a.y+dy*i/n;for(const[p,q,w,h]of walls)if(x>p-2&&x<p+w+2&&y>q-2&&y<q+h+2)return false}return true}
function walk(e,tx,ty,sp,dt){const dx=tx-e.x,dy=ty-e.y,d=Math.hypot(dx,dy);if(d<3)return true;const s=Math.min(sp*dt,d),mx=dx/d*s,my=dy/d*s;let ok=false;if(!hitWall(e.x+mx,e.y,R)){e.x+=mx;ok=true}if(!hitWall(e.x,e.y+my,R)){e.y+=my;ok=true}if(ok){e.mv=1;e.dx=mx;e.dz=my}return ok}
function spot(){for(;;){const x=30+Math.random()*(W-60),y=30+Math.random()*(H-60);if(!hitWall(x,y,R+8)&&ents.every(o=>Math.hypot(o.x-x,o.y-y)>50))return{x,y}}}
function wander(e){const p=spot();e.tx=p.x;e.ty=p.y;e.wait=Math.random()*1.5}
function toast(t){const el=$('toast');el.textContent=t;el.classList.add('on');clearTimeout(tt);tt=setTimeout(()=>el.classList.remove('on'),2200)}
function floorTex(m){const c=document.createElement('canvas');c.width=W;c.height=H;const x=c.getContext('2d');x.fillStyle=m.fl;x.fillRect(0,0,W,H);x.fillStyle=m.fl2;
 if(m.pat==='check'){for(let i=0;i<20;i++)for(let j=0;j<14;j++)if((i+j)%2==0)x.fillRect(i*40,j*40,40,40)}
 else if(m.pat==='lanes'){for(let y=160;y<560;y+=120)for(let k=0;k<W;k+=60)x.fillRect(k,y,30,4)}
 else{x.fillRect(0,250,W,60);x.fillRect(370,0,60,H);x.beginPath();x.arc(HX,HY,110,0,7);x.fill();x.fillStyle=m.fl;x.beginPath();x.arc(HX,HY,80,0,7);x.fill()}
 return new THREE.CanvasTexture(c)}
function label(t){const c=document.createElement('canvas');c.width=128;c.height=40;const x=c.getContext('2d');x.font='bold 24px sans-serif';x.textAlign='center';x.fillStyle='#fff';x.strokeStyle='rgba(0,0,0,.75)';x.lineWidth=5;x.strokeText(t,64,28);x.fillText(t,64,28);
 const s=new THREE.Sprite(new THREE.SpriteMaterial({map:new THREE.CanvasTexture(c),depthTest:false}));s.scale.set(48,15,1);s.position.y=50;s.renderOrder=5;return s}
function mkChar(e){const g=new THREE.Group(),inn=new THREE.Group();
 const b=new THREE.Mesh(gBody,new THREE.MeshLambertMaterial({color:new THREE.Color(`hsl(${e.hue},65%,55%)`)}));b.position.y=10;
 const h=new THREE.Mesh(gHead,mSkin);h.position.y=29;const v=new THREE.Mesh(gVis,mDark);v.position.set(0,30,6);
 inn.add(b,h,v);const bl=new THREE.Mesh(gBlob,mBlob);bl.rotation.x=-Math.PI/2;bl.position.y=.5;g.add(inn,bl);
 e.lab=label(e.name);g.add(e.lab);
 if(e.isPlayer){e.ring=new THREE.Mesh(gRingP,mWhite);e.ring.rotation.x=-Math.PI/2;e.ring.position.y=.8;g.add(e.ring)}
 e.g=g;e.inner=inn;e.body=b;world.add(g)}
function addRing(x,y,col,max,s0){const m=new THREE.Mesh(gFx,new THREE.MeshBasicMaterial({color:col,transparent:true,side:THREE.DoubleSide}));m.rotation.x=-Math.PI/2;m.position.set(x-HX,1.5,y-HY);world.add(m);fx.push({m,t:max,max,k:'r',s0})}
function addTracer(a,b){const d=D(a,b),m=new THREE.Mesh(new THREE.BoxGeometry(2,2,d),new THREE.MeshBasicMaterial({color:0xffd23f,transparent:true}));
 m.position.set((a.x+b.x)/2-HX,20,(a.y+b.y)/2-HY);m.lookAt(b.x-HX,20,b.y-HY);world.add(m);fx.push({m,t:.2,max:.2,k:'t'})}
function newRound(){
 if(world){world.traverse(o=>{if(o.material&&o.material.map)o.material.map.dispose()});scene.remove(world)}
 world=new THREE.Group();scene.add(world);const M=MAPS[mapIdx];walls=M.w;rd.setClearColor(M.bg);
 const fl=new THREE.Mesh(new THREE.PlaneGeometry(W,H),new THREE.MeshLambertMaterial({map:floorTex(M)}));fl.rotation.x=-Math.PI/2;world.add(fl);
 const mw=new THREE.MeshLambertMaterial({color:M.wl}),mb=new THREE.MeshLambertMaterial({color:new THREE.Color(M.wl).multiplyScalar(.6)});
 walls.forEach(([a,b,c,d])=>{const m=new THREE.Mesh(new THREE.BoxGeometry(c,M.h,d),mw);m.position.set(a+c/2-HX,M.h/2,b+d/2-HY);world.add(m)});
 [[0,-8,W,8],[0,H,W,8],[-8,0,8,H],[W,0,8,H]].forEach(([a,b,c,d])=>{const m=new THREE.Mesh(new THREE.BoxGeometry(c,M.h*.5,d),mb);m.position.set(a+c/2-HX,M.h/4,b+d/2-HY);world.add(m)});
 const roles=['m','s','i','i','i','i','i','i'].sort(()=>Math.random()-.5);
 ents=[];fx=[];gun=null;gunM=null;known=false;timer=ROUND;
 NAMES.forEach((n,i)=>{const p=spot();const e={name:n,x:p.x,y:p.y,role:roles[i],alive:true,isPlayer:i==0,hue:i*45,cd:0,rest:0,aim:0,wait:0,stuck:0,tx:p.x,ty:p.y,chase:null,wants:false,kb:0,mv:0,dx:0,dz:1};ents.push(e);mkChar(e)});
 player=ents[0];ents.slice(1).forEach(wander);
 [...$('maps').children].forEach((c,i)=>c.classList.toggle('sel',i==mapIdx));
 $('ot').textContent=INFO[player.role][0];$('op').textContent=INFO[player.role][1];$('ob').textContent='Start round';$('ov').hidden=false;state='intro'}
MAPS.forEach((m,i)=>{const b=document.createElement('button');b.className='chip';b.textContent=m.n;b.onclick=()=>{mapIdx=i;newRound()};$('maps').appendChild(b)});
$('ob').onclick=()=>{if(state==='end'){newRound();return}$('ov').hidden=true;state='play'};
function end(w,msg){if(state!=='play')return;state='end';const win=(player.role==='m')===(w==='m');
 setTimeout(()=>{$('ot').textContent=win?'You win! 🎉':'You lose';$('op').textContent=msg;$('ob').textContent='Play again';$('ov').hidden=false},900)}
function checkEnd(){const m=ents.find(o=>o.role==='m');if(!m.alive)end('i','The murderer was caught.');else if(!ents.some(o=>o.alive&&o.role!=='m'))end('m','Everyone was eliminated.')}
function kill(k,v){if(state!=='play'||!v.alive)return;v.alive=false;
 v.inner.rotation.set(-Math.PI/2,0,0);v.inner.position.y=10;v.lab.visible=false;if(v.ring)v.ring.visible=false;v.body.material.color.multiplyScalar(.55);
 if(k.role==='m'){addRing(v.x,v.y,0xff4d5e,.5,1.2);k.cd=.7;k.rest=1.5;for(const o of ents)if(o.alive&&o!==k&&!o.isPlayer&&D(o,k)<180&&los(o,k))known=true}
 if(v.role==='s'){gun={x:v.x,y:v.y};gunM=new THREE.Group();const a=new THREE.Mesh(new THREE.BoxGeometry(16,5,5),new THREE.MeshBasicMaterial({color:0xffd23f})),b=new THREE.Mesh(new THREE.BoxGeometry(5,9,5),new THREE.MeshBasicMaterial({color:0xffd23f}));b.position.set(-5,-5,0);gunM.add(a,b);gunM.position.set(v.x-HX,14,v.y-HY);world.add(gunM);
  ents.forEach(o=>o.wants=o.alive&&o.role==='i'&&!o.isPlayer&&Math.random()<.5);if(!v.isPlayer)toast('The sheriff fell — the gun dropped!')}
 else toast(v.isPlayer?'You were eliminated':v.name+' was eliminated');
 checkEnd()}
function shoot(s,t){addTracer(s,t);if(t.role==='m')kill(s,t);else{kill(s,t);if(s.alive){if(s.isPlayer)toast('You shot an innocent!');kill(s,s)}}}
function bot(e,dt){
 const m=ents.find(o=>o.role==='m'&&o.alive);e.cd-=dt;e.rest-=dt;let busy=false;
 if(e.role==='m'){if(e.rest<=0){let b=null,bd=330;for(const o of ents)if(o.alive&&o!==e){const d=D(e,o);if(d<bd&&los(e,o)){b=o;bd=d}}
  if(b){e.tx=b.x;e.ty=b.y;busy=true;if(bd<STAB&&e.cd<=0)kill(e,b)}}}
 else if(e.role==='s'){if(m&&known){const d=D(e,m);if(d<SHOOT&&los(e,m)){busy=true;e.tx=e.x;e.ty=e.y;e.aim+=dt;if(e.aim>.7&&e.cd<=0){e.aim=0;e.cd=2.5;if(Math.random()<.65)shoot(e,m);else addTracer(e,{x:m.x+30,y:m.y+20})}}else{e.aim=0;if(d<380){e.tx=m.x;e.ty=m.y;busy=true}}}}
 else{if(gun&&e.wants){e.tx=gun.x;e.ty=gun.y;busy=true}
  else if(m&&known){const d=D(e,m);if(d<200&&los(e,m)){const a=Math.atan2(e.y-m.y,e.x-m.x);e.tx=e.x+Math.cos(a)*120;e.ty=e.y+Math.sin(a)*120;busy=true}}}
 const ok=walk(e,e.tx,e.ty,busy?(e.role==='m'?92:95):70,dt);
 if(!busy){const d=Math.hypot(e.tx-e.x,e.ty-e.y);if(d<8){e.wait-=dt;if(e.wait<=0)wander(e)}else if(!ok){e.stuck+=dt;if(e.stuck>.4){wander(e);e.stuck=0}}}}
function update(dt){
 timer-=dt;if(timer<=0){end('i','Time ran out. The innocents survived.');return}
 const p=player;p.cd-=dt;
 if(p.alive){const vx=(keys.d||keys.ArrowRight?1:0)-(keys.a||keys.ArrowLeft?1:0),vy=(keys.s||keys.ArrowDown?1:0)-(keys.w||keys.ArrowUp?1:0);
  if(vx||vy){p.chase=null;p.kb=1;p.tx=p.x+vx*40;p.ty=p.y+vy*40}else if(p.kb){p.kb=0;p.tx=p.x;p.ty=p.y}
  else if(p.chase&&p.chase.alive){p.tx=p.chase.x;p.ty=p.chase.y;if(D(p,p.chase)<STAB&&p.cd<=0)kill(p,p.chase)}
  walk(p,p.tx,p.ty,100,dt)}
 for(const e of ents)if(e.alive&&!e.isPlayer)bot(e,dt);
 if(gun)for(const e of ents)if(e.alive&&e.role==='i'&&Math.hypot(e.x-gun.x,e.y-gun.y)<18){e.role='s';gun=null;world.remove(gunM);gunM=null;toast(e.isPlayer?'You picked up the gun!':e.name+' grabbed the gun');break}
 fx.forEach(f=>f.t-=dt);fx=fx.filter(f=>{if(f.t>0)return true;world.remove(f.m);f.m.material.dispose();return false})}
function act(t){if(state!=='play'||!player.alive||player.role==='i')return;
 if(!t){let bd=1e9;for(const o of ents)if(o.alive&&o!==player&&los(player,o)){const d=D(player,o);if(d<bd){bd=d;t=o}}if(!t)return}
 const d=D(player,t);
 if(player.role==='m'){if(d<=STAB&&player.cd<=0)kill(player,t);else player.chase=t}
 else{if(player.cd>0){toast('Reloading…');return}if(d>SHOOT||!los(player,t)){toast('Out of range');return}player.cd=2;shoot(player,t)}}
$('act').onclick=()=>act();
const ray=new THREE.Raycaster(),pl=new THREE.Plane(new THREE.Vector3(0,1,0),0),v3=new THREE.Vector3();
cv.addEventListener('pointerdown',ev=>{if(state!=='play'||!player.alive)return;const r=cv.getBoundingClientRect(),px=ev.clientX-r.left,py=ev.clientY-r.top;
 let t=null,bd=r.width*.045;for(const o of ents)if(o.alive&&o!==player){v3.set(o.x-HX,16,o.y-HY).project(cam);const d=Math.hypot((v3.x+1)/2*r.width-px,(1-v3.y)/2*r.height-py);if(d<bd){bd=d;t=o}}
 if(t&&player.role!=='i'){act(t);return}
 ray.setFromCamera({x:px/r.width*2-1,y:-(py/r.height)*2+1},cam);
 if(ray.ray.intersectPlane(pl,v3)){const x=v3.x+HX,y=v3.z+HY;player.chase=null;player.kb=0;player.tx=x;player.ty=y;addRing(x,y,0x7c8cff,.5,.5)}});
addEventListener('keydown',e=>{const k=e.key.length===1?e.key.toLowerCase():e.key;keys[k]=1;if(k===' '||k.startsWith('Arrow'))e.preventDefault();if(k===' ')act()});
addEventListener('keyup',e=>{keys[e.key.length===1?e.key.toLowerCase():e.key]=0});
function sync(t){
 for(const e of ents){e.g.position.set(e.x-HX,0,e.y-HY);
  if(e.alive){e.inner.position.y=e.mv?Math.abs(Math.sin(t*.012))*2:0;if(e.mv)e.inner.rotation.y=Math.atan2(e.dx,e.dz);e.mv=0}}
 if(gunM){gunM.position.y=14+Math.sin(t*.005)*3;gunM.rotation.y+=.03}
 for(const f of fx){const p=1-f.t/f.max;if(f.k==='r'){f.m.scale.setScalar(f.s0*(1+p*3));f.m.material.opacity=1-p}else f.m.material.opacity=Math.min(1,f.t*5)}}
function resize(){rd.setSize(cv.clientWidth,cv.clientWidth*H/W,false)}
addEventListener('resize',resize);
let last=performance.now();
function loop(now){const dt=Math.min(.05,(now-last)/1000);last=now;if(state==='play')update(dt*(player.alive?1:2.5));sync(now);rd.render(scene,cam);
 const rn={m:'🔪 Murderer',s:'🔫 Sheriff',i:'🙂 Innocent'};$('role').textContent=rn[player.role]+(player.alive?'':' (out)');
 const t=Math.max(0,Math.ceil(timer));$('time').textContent=Math.floor(t/60)+':'+String(t%60).padStart(2,'0');
 $('alive').textContent='Alive '+ents.filter(o=>o.alive).length;
 const b=$('act'),L={m:'🔪 Stab nearest',s:'🔫 Shoot nearest',i:'Survive until time runs out'}[player.role];if(b.textContent!==L)b.textContent=L;b.disabled=player.role==='i'||!player.alive;
 requestAnimationFrame(loop)}
resize();newRound();requestAnimationFrame(loop);
</script></body></html>
