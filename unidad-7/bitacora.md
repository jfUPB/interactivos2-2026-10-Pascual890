# Unidad 7

## Bitácora de proceso de aprendizaje

¿Qué parámetros de tu obra se van a controlar en tiempo real? (parámetros de audio, de visuales, o ambos)

Van a cambiar tanto visuales como audio. Por medio de movimiento de elementos con la mano, el sistema cambiará parámetros de colores, volumen, octavas y ritmo de los drums.

¿Quién controla qué? (¿Qué controla el artista/performer y qué controla el público?)

El público maneja todo. Con la detección de mano de ml5 y la cámara del pc se va a controlar todo. Yo como artista solo tengo que modificar parametros del strudel si quiero, como el bpm para que sea mas lento o rápido, pero no esnecesario hacer nada en vivo.

¿Cómo participará el público? (¿Con el celular?, ¿Con gestos?, ¿Con sensores?, ¿Qué datos enviarán?)

Con gestos. Por medio del modelo usado de ml5, se va a trackear la mano para usar gestos que cambien la escena. Para poder modificar el audio moviendo las visuales, se usan señales midi.

¿Qué dispositivos se usarán? (Open Stage Control, celular vía Socket.io, controladores MIDI, sensores, etc.)

La cámara del laptop para mandar imagen al modelo y tomar los datos de la posicion de los dedos, loopMIDI para el midi virtual que conectará al strudel.

## Bitácora de aplicación 

Visuales y programa
``` html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <title>TUI</title>
  <style>
    * { margin: 0; padding: 0; box-sizing: border-box; }
    body { background: #12141a; overflow: hidden; font-family: 'Georgia', serif; }
    canvas { display: block; position: fixed; top: 0; left: 0; }
    #hud {
      position: fixed; top: 0; left: 0; right: 0; bottom: 0;
      pointer-events: none;
      display: flex; flex-direction: column;
      justify-content: space-between;
      padding: 28px 36px;
      z-index: 10;
    }
    #top-row { display: flex; justify-content: space-between; align-items: flex-start; }
    #gesture-state {
      font-size: 12px; letter-spacing: 0.22em;
      color: rgba(255,255,255,0.55);
      font-style: italic;
      transition: color 0.5s ease;
    }
    #orb-legend {
      display: flex; flex-direction: column; align-items: flex-end; gap: 8px;
    }
    .legend-row {
      display: flex; align-items: center; gap: 9px;
      font-size: 9px; letter-spacing: 0.25em;
      color: rgba(255,255,255,0.35); text-transform: uppercase;
    }
    .legend-swatch { width: 18px; height: 1px; }
    #bottom-row { display: flex; justify-content: space-between; align-items: flex-end; }
    #ws-dot { font-size: 9px; letter-spacing: 0.22em; color: rgba(255,255,255,0.25); }
    #hint { font-size: 8px; letter-spacing: 0.2em; color: rgba(255,255,255,0.16); text-align: right; }
  </style>
</head>
<body>
<div id="hud">
  <div id="top-row">
    <div id="gesture-state">escuchando…</div>
    <div id="orb-legend">
      <div class="legend-row"><span class="legend-swatch" style="background:rgba(255,80,60,0.9)"></span>kick · sd</div>
      <div class="legend-row"><span class="legend-swatch" style="background:rgba(255,210,50,0.9)"></span>hats</div>
      <div class="legend-row"><span class="legend-swatch" style="background:rgba(50,180,255,0.9)"></span>bass · piano</div>
      <div class="legend-row"><span class="legend-swatch" style="background:rgba(200,80,255,0.9)"></span>arp · square</div>
    </div>
  </div>
  <div id="bottom-row">
    <div id="ws-dot">— —</div>
    <div id="hint">puño → atraer todos · pellizco → mover uno</div>
  </div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/p5.js/1.9.3/p5.min.js"></script>
<script src="https://unpkg.com/ml5@1/dist/ml5.min.js"></script>
<script>
// ════════════════════════════════════════════════════════
// ORBES
// ════════════════════════════════════════════════════════
const ORB_CONFIG = [
  { id:'KICK_SD', label:'kick · sd',  sublabel:'vel · vol', targets:['bd','sd','cp','rim'],  h:5,   s:85, b:90, rings:4, pulse:0, expand:0 },
  { id:'HATS',    label:'hats',       sublabel:'vel · vol', targets:['hh','oh','cy'],        h:48,  s:90, b:95, rings:3, pulse:0, expand:0 },
  { id:'BASS',    label:'bass',       sublabel:'oct · vol', targets:['piano'],     h:200, s:80, b:85, rings:5, pulse:0, expand:0 },
  { id:'ARP',     label:'arp',        sublabel:'atm · vel', targets:['square'],    h:290, s:75, b:88, rings:5, pulse:0, expand:0 }
];

let orbs    = ORB_CONFIG.map((cfg, i) => ({ ...cfg, x: 0.2 + i * 0.2, y: 0.5 }));
let bgBlobs = [];
let bgPulse = 0;
let ws;

let bgHue = 220, bgSat = 28, bgBri = 12;
let targetBgHue = 220, targetBgSat = 28, targetBgBri = 12;

// ════════════════════════════════════════════════════════
// WEBSOCKET
// ════════════════════════════════════════════════════════
function connectBridge() {
  ws = new WebSocket('ws://127.0.0.1:8081');
  ws.onopen  = () => { document.getElementById('ws-dot').textContent = '● en vivo'; };
  ws.onclose = () => {
    document.getElementById('ws-dot').textContent = '○ desconectado';
    setTimeout(connectBridge, 3000);
  };
  ws.onmessage = (e) => {
    try {
      const msg = JSON.parse(e.data);
      if (!msg.address || !msg.address.includes('play')) return;

      bgPulse = min(bgPulse + 0.3, 1.0);
      const args = msg.args || [];
      let sample = '';

      for (let i = 0; i < args.length; i++) {
        const a = args[i];
        if (typeof a === 'object' && a !== null && a.key === 's') { sample = String(a.value || ''); break; }
        if (a === 's' && i + 1 < args.length) {
          const next = args[i+1];
          sample = typeof next === 'object' ? String(next.value || '') : String(next || '');
          break;
        }
      }

      if (sample) {
        orbs.forEach(orb => {
          if (orb.targets.some(t => sample.toLowerCase().startsWith(t))) {
            orb.pulse  = 1.0;
            orb.expand = 1.0;
          }
        });
      }
    } catch(_) {}
  };
}
connectBridge();

let handPose, video, hands = [];
function preload() { handPose = ml5.handPose({ maxHands: 1, flipped: true }); }

// ════════════════════════════════════════════════════════
// SETUP
// ════════════════════════════════════════════════════════
function setup() {
  createCanvas(windowWidth, windowHeight);
  colorMode(HSB, 360, 100, 100, 1.0);

  for (let i = 0; i < 7; i++) {
    bgBlobs.push({
      x: random(width), y: random(height),
      vx: random(-0.4, 0.4), vy: random(-0.3, 0.3),
      baseR: random(130, 300),
      h: random(190, 260), s: 35, b: 28,
      phase: random(TWO_PI), noiseOff: random(100)
    });
  }

  video = createCapture(VIDEO);
  video.size(640, 480);
  video.hide();
  handPose.detectStart(video, r => { hands = r; });
}

function windowResized() { resizeCanvas(windowWidth, windowHeight); }

// ════════════════════════════════════════════════════════
// DRAW
// ════════════════════════════════════════════════════════
function draw() {
  updateBgColor();
  background(bgHue, bgSat, bgBri, 0.22);

  drawBgBlobs();
  drawBgIllum();
  drawOrbConnections();

  // Gestos
  if (hands.length > 0) {
    const kp     = hands[0].keypoints;
    const thumb  = mapKP(kp[4]);
    const index  = mapKP(kp[8]);
    const pinchX = (thumb.x + index.x) / 2;
    const pinchY = (thumb.y + index.y) / 2;
    const normPX = pinchX / width;
    const normPY = pinchY / height;

    let dSum = [8,12,16,20].reduce((a,i) =>
      a + dist(kp[i].x, kp[i].y, kp[0].x, kp[0].y), 0) / 4;
    const isFist  = dSum < 65;
    const isPinch = dist(thumb.x, thumb.y, index.x, index.y) < 55 && !isFist;

    if (isFist) {
      orbs.forEach(o => { o.x = lerp(o.x, normPX, 0.06); o.y = lerp(o.y, normPY, 0.06); });
      drawMagnetEffect(pinchX, pinchY);
      document.getElementById('gesture-state').textContent = 'atrayendo';
      document.getElementById('gesture-state').style.color = 'rgba(200,160,255,0.9)';
    } else if (isPinch) {
      let closest = orbs.reduce((prev, curr) =>
        dist(pinchX, pinchY, curr.x*width, curr.y*height) <
        dist(pinchX, pinchY, prev.x*width, prev.y*height) ? curr : prev);
      closest.x = lerp(closest.x, normPX, 0.25);
      closest.y = lerp(closest.y, normPY, 0.25);
      document.getElementById('gesture-state').textContent = `modificando ${closest.label}`;
      document.getElementById('gesture-state').style.color = 'rgba(220,190,100,0.9)';
    } else {
      document.getElementById('gesture-state').textContent = 'escuchando…';
      document.getElementById('gesture-state').style.color = 'rgba(255,255,255,0.55)';
    }

    if (frameCount % 3 === 0 && ws?.readyState === 1)
      ws.send(JSON.stringify({ type: 'orbs_update', orbs }));

    let hAura, sAura, bAura;
    if (isFist)       { hAura=290; sAura=50; bAura=90; }
    else if (isPinch) { hAura=45;  sAura=85; bAura=95; }
    else              { hAura=180; sAura=40; bAura=80; }
    drawHandSkeleton(kp.map(mapKP), isPinch, hAura, sAura, bAura);
  }

  orbs.forEach(o => drawOrb(o));

  // Decaimientos al final
  orbs.forEach(o => { o.pulse *= 0.82; o.expand *= 0.72; });
  bgPulse *= 0.92;
}

// ════════════════════════════════════════════════════════
// COLOR DE FONDO DINÁMICO
// ════════════════════════════════════════════════════════
function updateBgColor() {
  let bassY = map(orbs[2].y, 0, 1, 0, 1);
  let arpX  = map(orbs[3].x, 0, 1, 0, 1);
  targetBgHue = 220 + bassY * 60 + arpX * 20;
  targetBgSat = 22 + bassY * 20;
  targetBgBri = 10 + map(orbs[0].x, 0, 1, 0, 5);
  bgHue = lerp(bgHue, targetBgHue, 0.012);
  bgSat = lerp(bgSat, targetBgSat, 0.015);
  bgBri = lerp(bgBri, targetBgBri, 0.018);
}

// ════════════════════════════════════════════════════════
// BLOBS DE FONDO
// ════════════════════════════════════════════════════════
function drawBgBlobs() {
  let avgY  = orbs.reduce((a, b) => a + b.y, 0) / orbs.length;
  let speed = map(avgY, 0, 1, 0.4, 4.0);
  let t     = frameCount * 0.01 * speed;

  bgBlobs.forEach((b, i) => {
    b.x += b.vx * speed;
    b.y += b.vy * speed;
    if (b.x < -200 || b.x > width  + 200) b.vx *= -1;
    if (b.y < -200 || b.y > height + 200) b.vy *= -1;

    let hue   = (b.h + map(orbs[2].y, 0, 1, -40, 120)) % 360;
    let sat   = b.s + map(orbs[3].x, 0, 1, 0, 22);
    let alpha = map(orbs[0].x, 0, 1, 0.08, 0.26);
    let r     = b.baseR * 0.85;

    noStroke();
    fill(hue, sat, b.b, alpha);
    beginShape();
    for (let a = 0; a < TWO_PI; a += 0.55) {
      let rr = r * (1 + noise(cos(a) + i, sin(a) + t) * 0.4);
      curveVertex(b.x + cos(a)*rr, b.y + sin(a)*rr);
    }
    endShape(CLOSE);
  });
}

// ════════════════════════════════════════════════════════
// ILUMINACIÓN POR POSICIÓN DE ORBES
// ════════════════════════════════════════════════════════
function drawBgIllum() {
  noStroke();
  orbs.forEach(orb => {
    const px = orb.x * width;
    const py = orb.y * height;
    const illumR    = width * (0.22 + orb.y * 0.15);
    const baseAlpha = 0.025 + orb.expand * 0.025 + orb.pulse * 0.02;
    let hue = (orb.h + map(orb.y, 0, 1, -15, 30)) % 360;
    for (let r = 4; r >= 1; r--) {
      fill(hue, orb.s * 0.6, 55 + orb.pulse * 20, baseAlpha * (1 - r/5) * 2.2);
      ellipse(px, py, illumR*(r/4)*2, illumR*(r/4)*2);
    }
  });
}

// ════════════════════════════════════════════════════════
// CONEXIONES ENTRE ORBES
// ════════════════════════════════════════════════════════
function drawOrbConnections() {
  noFill();
  const t = frameCount * 0.008;
  for (let i = 0; i < orbs.length; i++) {
    for (let j = i+1; j < orbs.length; j++) {
      const a=orbs[i], b=orbs[j];
      const ax=a.x*width, ay=a.y*height, bx=b.x*width, by=b.y*height;
      const d=dist(ax,ay,bx,by);
      if (d > width*0.65) continue;
      stroke((a.h+b.h)/2,(a.s+b.s)*0.4,70,map(d,0,width*0.65,0.10,0.01)+(a.pulse+b.pulse)*0.06);
      strokeWeight(map(d,0,width*0.65,1.2,0.3));
      const mx=(ax+bx)/2+noise(i*3,j*7,t)*50-25;
      const my=(ay+by)/2+noise(i*5,j*9,t)*50-25;
      beginShape();
      curveVertex(ax,ay); curveVertex(ax,ay);
      curveVertex(mx,my);
      curveVertex(bx,by); curveVertex(bx,by);
      endShape();
    }
  }
}

// ════════════════════════════════════════════════════════
// ORBES — reacción limpia al audio sin efectos de partículas
// ════════════════════════════════════════════════════════
function drawOrb(orb) {
  const px = orb.x * width;
  const py = orb.y * height;
  const p  = orb.pulse;
  const ex = orb.expand;
  const t  = frameCount * 0.018;

  let extraSize=0, satMod=0, brightMod=0, auraScale=1;
  if (orb.id==='KICK_SD') { satMod=map(orb.y,0,1,-8,15); extraSize=map(orb.y,0,1,-6,6); }
  else if (orb.id==='HATS') { auraScale=map(orb.y,0,1,1.2,0.75); satMod=map(orb.y,0,1,0,12); }
  else if (orb.id==='BASS') { extraSize=map(orb.y,0,1,-14,18); brightMod=map(orb.y,0,1,10,-6); satMod=map(orb.y,0,1,-8,12); }
  else if (orb.id==='ARP')  { auraScale=map(orb.x,0,1,0.75,1.5); satMod=map(orb.x,0,1,-4,10); }

  const BASE_SIZE   = 32;
  const expandBoost = ex * 38; // solo grande cuando hay hit real

  push();
  translate(px, py);
  noStroke();

  // Capas de acuarela
  for (let j = orb.rings; j >= 1; j--) {
    const baseR  = (BASE_SIZE + j*14 + extraSize + expandBoost) * auraScale;
    const alpha  = map(j, 1, orb.rings, 0.14 + p*0.22, 0.03 + p*0.04);
    const hShift = sin(t*0.7 + j) * 10;
    fill(
      orb.h + hShift,
      constrain(orb.s + satMod, 20, 100) * (0.55 + p*0.45),
      constrain(orb.b + brightMod, 25, 100) * (0.65 + p*0.35),
      alpha
    );
    beginShape();
    for (let i = 0; i <= 24; i++) {
      const angle = (i/24)*TWO_PI;
      const nv    = noise(cos(angle)*0.9 + j*3.1, sin(angle)*0.9 + orb.h*0.01, t*0.5);
      const offset = map(nv, 0, 1, -baseR*0.25, baseR*0.25) * (0.35 + p*0.65);
      curveVertex(cos(angle)*(baseR+offset), sin(angle)*(baseR+offset));
    }
    endShape(CLOSE);
  }

  // Núcleo — crece con el hit
  const coreR     = (8 + extraSize*0.25) * max(0.5, auraScale*0.85);
  const coreBurst = coreR + p*10 + ex*14;
  for (let layer = 3; layer >= 1; layer--) {
    fill(orb.h, orb.s, orb.b*(0.9+p*0.1), 0.13*layer + p*0.18);
    circle(0, 0, coreBurst*layer*1.3);
  }
  fill(orb.h, orb.s*0.6, min(orb.b+22,100), 0.85+p*0.15);
  circle(0, 0, coreBurst);

  // Anillo de expansión al hit — solo cuando expand es significativo
  if (ex > 0.08) {
    noFill();
    const ringR = BASE_SIZE + expandBoost*1.3 + extraSize;
    stroke(orb.h, orb.s*0.8, min(orb.b+15,100), ex*0.60);
    strokeWeight(1.2 + ex*2.2);
    beginShape();
    for (let i = 0; i <= 28; i++) {
      const angle = (i/28)*TWO_PI;
      const nv = noise(cos(angle)*0.4+orb.h*0.02, sin(angle)*0.4, frameCount*0.06);
      curveVertex(
        cos(angle)*ringR*map(nv,0,1,0.93,1.07),
        sin(angle)*ringR*map(nv,0,1,0.93,1.07)
      );
    }
    endShape(CLOSE);

    // Segundo anillo tenue
    if (ex > 0.25) {
      stroke(orb.h, orb.s*0.4, orb.b+8, ex*0.16);
      strokeWeight(0.6);
      beginShape();
      for (let i = 0; i <= 24; i++) {
        const angle = (i/24)*TWO_PI;
        curveVertex(cos(angle)*ringR*1.4, sin(angle)*ringR*1.4);
      }
      endShape(CLOSE);
    }
  }

  // Etiqueta
  push();
  colorMode(RGB, 255, 255, 255, 1.0);
  fill(255, 255, 255, 0.65 + p*0.35);
  noStroke();
  textFont('Georgia');
  textStyle(ITALIC);
  textSize(11);
  textAlign(CENTER);
  text(orb.label, 0, coreBurst/2 + 20);
  fill(255, 255, 255, 0.32 + p*0.20);
  textStyle(NORMAL);
  textSize(8);
  text(orb.sublabel, 0, coreBurst/2 + 33);
  colorMode(HSB, 360, 100, 100, 1.0);
  pop();

  pop();
}

// ════════════════════════════════════════════════════════
// EFECTO MAGNÉTICO
// ════════════════════════════════════════════════════════
function drawMagnetEffect(x, y) {
  noFill();
  stroke(290, 60, 95, 0.5);
  strokeWeight(1.5);
  circle(x, y, (frameCount*6) % 200);
}

// ════════════════════════════════════════════════════════
// ESQUELETO DE MANO
// ════════════════════════════════════════════════════════
function drawHandSkeleton(kp, isPinch, hAura, sAura, bAura) {
  push();
  const chains = [[0,1,2,3,4],[0,5,6,7,8],[9,10,11,12],[13,14,15,16],[0,17,18,19,20],[5,9,13,17]];
  noFill();
  stroke(hAura, sAura, bAura, 0.40);
  strokeWeight(1.5);
  for (const chain of chains) {
    beginShape();
    for (const idx of chain) curveVertex(kp[idx].x, kp[idx].y);
    endShape();
  }
  noStroke();
  for (let i = 0; i < kp.length; i++) {
    const isKey = i===4||i===8, isTip=[4,8,12,16,20].includes(i);
    fill(hAura, sAura, bAura, isKey ? 0.70 : 0.38);
    circle(kp[i].x, kp[i].y, isKey ? 10 : (isTip ? 6 : 4));
  }
  if (isPinch) {
    stroke(hAura, sAura*0.6, bAura+5, 0.35);
    strokeWeight(0.8);
    drawingContext.setLineDash([2, 7]);
    line(kp[4].x, kp[4].y, kp[8].x, kp[8].y);
    drawingContext.setLineDash([]);
  }
  pop();
}

function mapKP(p) { return { x: map(p.x,0,640,0,width), y: map(p.y,0,480,0,height) }; }
</script>
</body>
</html>


```

Bridge

``` js
const WebSocket = require('ws');
const easymidi  = require('easymidi');

const wssP5      = new WebSocket.Server({ port: 8081 });
const wssStrudel = new WebSocket.Server({ port: 8080 });
const midiOut    = new easymidi.Output('StrudelMIDI');

let p5Clients = [];

wssP5.on('connection', (ws) => {
  p5Clients.push(ws);
  console.log(`[p5] conectado. Total: ${p5Clients.length}`);

  ws.on('message', (msg) => {
    try {
      const data = JSON.parse(msg);
      if (data.type !== 'orbs_update') return;
      const ORB_MAP = [
        { ccX:10, ccY:11 }, { ccX:12, ccY:13 },
        { ccX:14, ccY:15 }, { ccX:16, ccY:17 }
      ];
      data.orbs.forEach((orb, i) => {
        if (i >= ORB_MAP.length) return;
        midiOut.send('cc', { controller: ORB_MAP[i].ccX, value: Math.round(orb.x * 127), channel: 0 });
        midiOut.send('cc', { controller: ORB_MAP[i].ccY, value: Math.round(orb.y * 127), channel: 0 });
      });
    } catch(e) {}
  });

  ws.on('close', () => {
    p5Clients = p5Clients.filter(c => c !== ws);
  });
});

wssStrudel.on('connection', (ws) => {
  console.log('[Strudel] conectado al puerto 8080');

  ws.on('message', (msg) => {
    const raw = msg.toString();

    // LOG — muestra los primeros 300 chars de cada mensaje de Strudel
    console.log('[Strudel msg]', raw.slice(0, 300));

    // Reenviar al p5
    p5Clients.forEach(client => {
      if (client.readyState === WebSocket.OPEN) {
        client.send(raw);
      }
    });
  });

  ws.on('close', () => console.log('[Strudel] desconectado'));
});

console.log('🔍 Bridge DEBUG activo — p5:8081 | Strudel:8080');
console.log('   Mirá la consola para ver el formato exacto de los mensajes OSC');
```
LoopMIDI

<img width="728" height="480" alt="Screenshot 2026-05-14 071643" src="https://github.com/user-attachments/assets/d00b6df1-d379-4298-b3cb-df73fe85afc2" />

## Bitácora de reflexión






