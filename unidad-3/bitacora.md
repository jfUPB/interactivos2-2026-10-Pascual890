# Unidad 3

## Bitácora de proceso de aprendizaje


## Bitácora de aplicación 

El código de strudel es el mismo que tenía anteriormente para trabajar con osc:

```js
setcps(0.75)

let base = "<d2 g1>/2" 
let notas = "<[f3,a3,c4] [bb3,d3,f4]>/2" 

let drumLayer1 = s("bd [~ bd] <bd!3 [bd*2]> ~").gain(1.2)
let drumLayer2 = s("~ sd").gain(0.8)
let drumLayer3 = s("hh*4").gain(0.7).velocity(sine.range(0.2, 0.9).slow(1)) 

let pianoLayer = note(stack(base, notas)).s("gm_epiano1")
  .gain(0.2)
  .room(3)
  .lpf(sine.range(500, 1500).slow(8))

let bajoLayer = note(base).s("gm_acoustic_bass")
  .lpf(800)
  .gain(1)

$: stack(
  drumLayer1,
  drumLayer2,
  drumLayer3,
  pianoLayer,
  bajoLayer,
  
  drumLayer1.osc(),
  drumLayer2.osc(),
  drumLayer3.osc(),
  pianoLayer.osc()
)
```

Luego se modificó el archivo de visuales y la sesion de open stage control con el fader y el pan:

```html
<!DOCTYPE html>
<html>
<head>
  <script src="https://cdn.jsdelivr.net/npm/p5@1.11.11/lib/p5.min.js"></script>
  <style> body { margin: 0; overflow: hidden; background: black; } </style>
</head>
<body>
<script>

  let eventQueue = [];
  let activeAnimations = [];
  const LATENCY_CORRECTION = 0;

  // PARÁMETROS CONTROLABLES DESDE OPEN STAGE CONTROL
  let ctrl = {
    bomboSize:    1.0,   // fader  /ctrl/bombo_size  (0.1 – 2.5)
    velocidad:    1.0,   // fader  /ctrl/velocidad   (0.1 – 3.0)
    colorHue:     0.0,   // xy X   /ctrl/xy           (0 – 360)
    glowAmount:   30,    // xy Y   /ctrl/xy           (5 – 80)
  };

  function setup() {
    createCanvas(windowWidth, windowHeight);
    rectMode(CENTER);
    noStroke();

    const socket  = new WebSocket('ws://localhost:8081'); // Bridge 1: Strudel
    const socket2 = new WebSocket('ws://localhost:8082'); // Bridge 2: OSC

    socket.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      let params = {};
      for (let i = 0; i < msg.args.length; i += 2) {
        params[msg.args[i]] = msg.args[i + 1];
      }
      eventQueue.push({
        timestamp: msg.timestamp,
        sound:     params.s,
        delta:     params.delta    || 0.25,
        velocity:  params.velocity || 0.7,
        params:    params
      });
      eventQueue.sort((a, b) => a.timestamp - b.timestamp);
    };

    socket2.onmessage = (event) => {
      const msg = JSON.parse(event.data);
      console.log('OSC recibido:', msg.address, msg.args);

      if (msg.address && msg.address.startsWith('/ctrl/')) {
        const param = msg.address.replace('/ctrl/', '');
        const val   = msg.args[0];
        aplicarControl(param, val, msg);  // <-- pasa msg completo
        return;
      }

      // Mensajes musicales de Strudel por este socket
      let params = {};
      for (let i = 0; i < msg.args.length; i += 2) {
        params[msg.args[i]] = msg.args[i + 1];
      }
      if (params.s) {
        eventQueue.push({
          timestamp: msg.timestamp || Date.now(),
          sound:     params.s,
          delta:     params.delta    || 0.25,
          velocity:  params.velocity || 0.7,
          params:    params
        });
        eventQueue.sort((a, b) => a.timestamp - b.timestamp);
      }
    };

    socket.onopen   = () => console.log('Conectado a Bridge 1 (Strudel)');
    socket2.onopen  = () => console.log('Conectado a Bridge 2 (OSC)');
    socket.onerror  = () => console.warn('Bridge 1 no disponible — ¿está corriendo?');
    socket2.onerror = () => console.warn('Bridge 2 no disponible — ¿está corriendo?');
  }

  // APLICA PARÁMETROS RECIBIDOS DESDE OSC
  function aplicarControl(param, val, msg) {
    switch (param) {
      case 'bombo_size': ctrl.bomboSize  = val;          break;
      case 'velocidad':  ctrl.velocidad  = val;          break;
      case 'color_hue':  ctrl.colorHue   = val;          break;
      case 'glow':       ctrl.glowAmount = val;          break;
      case 'xy':
        ctrl.colorHue   = msg.args[0];  // eje X → hue (0–360)
        ctrl.glowAmount = msg.args[1];  // eje Y → glow (5–80)
        break;
      default:
        console.log('Parámetro OSC desconocido:', param, val);
    }
    mostrarHUD = true;
    hudTimer   = 120;
  }

  let mostrarHUD = false;
  let hudTimer   = 0;


  function draw() {
    background(0, ctrl.glowAmount);

    let now = Date.now() + LATENCY_CORRECTION;

    while (eventQueue.length > 0 && now >= eventQueue[0].timestamp) {
      let ev = eventQueue.shift();
      activeAnimations.push({
        startTime: ev.timestamp,
        duration:  ev.delta * 1000 * ctrl.velocidad,
        type:      ev.sound,
        velocity:  ev.velocity,
        x: random(width * 0.2, width * 0.8),
        y: random(height * 0.2, height * 0.8),
        color: getColorForSound(ev.sound)
      });
    }

    for (let i = activeAnimations.length - 1; i >= 0; i--) {
      let anim     = activeAnimations[i];
      let elapsed  = now - anim.startTime;
      let progress = elapsed / anim.duration;

      if (progress <= 1.0) {
        dibujarElemento(anim, progress);
      } else {
        activeAnimations.splice(i, 1);
      }
    }

    if (mostrarHUD && hudTimer > 0) {
      dibujarHUD();
      hudTimer--;
      if (hudTimer <= 0) mostrarHUD = false;
    }
  }

  // HUD
  function dibujarHUD() {
    push();
    let alpha = map(hudTimer, 0, 60, 0, 200);
    fill(0, 0, 0, alpha * 0.6);
    noStroke();
    rectMode(CORNER);
    rect(width - 240, 10, 230, 115, 8);

    fill(255, 255, 255, alpha);
    textSize(12);
    textAlign(LEFT);
    text('🎛  OPEN STAGE CONTROL',                    width - 228, 30);
    text(`Bombo size : ${ctrl.bomboSize.toFixed(2)}`,  width - 228, 50);
    text(`Velocidad  : ${ctrl.velocidad.toFixed(2)}`,  width - 228, 65);
    text(`Color hue  : ${ctrl.colorHue.toFixed(0)}°`,  width - 228, 80);
    text(`Glow       : ${ctrl.glowAmount.toFixed(0)}`, width - 228, 95);
    pop();
  }

  // DIBUJO DE ELEMENTOS
  function dibujarElemento(anim, p) {
    push();
    const color = aplicarHue(anim.color);
    switch (anim.type) {
      case 'bd':          dibujarBombo(p, color);         break;
      case 'sd':          dibujarCaja(p, color);          break;
      case 'hh':          dibujarHat(anim, p, color);     break;
      case 'gm_epiano1':  dibujarPiano(p, color);         break;
      default:            dibujarDefault(anim, p, color); break;
    }
    pop();
  }

  function aplicarHue(c) {
    if (ctrl.colorHue === 0) return c;
    let r = c[0] / 255, g = c[1] / 255, b = c[2] / 255;
    let max = Math.max(r, g, b), min = Math.min(r, g, b);
    let h, s, l = (max + min) / 2;
    if (max === min) { h = s = 0; }
    else {
      let d = max - min;
      s = l > 0.5 ? d / (2 - max - min) : d / (max + min);
      switch (max) {
        case r: h = ((g - b) / d + (g < b ? 6 : 0)) / 6; break;
        case g: h = ((b - r) / d + 2) / 6;               break;
        case b: h = ((r - g) / d + 4) / 6;               break;
      }
    }
    h = (h + ctrl.colorHue / 360) % 1;
    function hue2rgb(p, q, t) {
      if (t < 0) t += 1; if (t > 1) t -= 1;
      if (t < 1/6) return p + (q - p) * 6 * t;
      if (t < 1/2) return q;
      if (t < 2/3) return p + (q - p) * (2/3 - t) * 6;
      return p;
    }
    let q = l < 0.5 ? l * (1 + s) : l + s - l * s;
    let p2 = 2 * l - q;
    return [
      Math.round(hue2rgb(p2, q, h + 1/3) * 255),
      Math.round(hue2rgb(p2, q, h)       * 255),
      Math.round(hue2rgb(p2, q, h - 1/3) * 255)
    ];
  }

  function dibujarBombo(p, c) {
    let bounce = sin(p * PI);
    let maxD   = 400 * ctrl.bomboSize;
    let d      = lerp(80 * ctrl.bomboSize, maxD, bounce);
    let alpha  = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    circle(width / 2, height / 2, d);
  }

  function dibujarCaja(p, c) {
    let w     = lerp(width, 0, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    rect(width / 2, height / 2, w, 50);
  }

  function dibujarHat(anim, p, c) {
    let maxHeight     = map(anim.velocity, 0.2, 0.9, 100, 300);
    let currentHeight = lerp(0, maxHeight, p);
    let yPos          = anim.y - currentHeight;
    let alpha         = lerp(255, 0, p);
    let thickness     = lerp(8, 2, p);

    stroke(c[0], c[1], c[2], alpha);
    strokeWeight(thickness);
    line(anim.x, anim.y, anim.x, yPos);

    noStroke();
    fill(255, 255, 255, alpha);
    circle(anim.x, yPos, thickness * 2);
  }

  function dibujarPiano(p, c) {
    noFill();
    stroke(c[0], c[1], c[2], lerp(200, 0, p));
    strokeWeight(3);
    let d = lerp(50, 300, p);
    circle(width / 2, height / 2, d);
  }

  function dibujarDefault(anim, p, c) {
    let size  = lerp(100, 0, p);
    let angle = p * TWO_PI;
    translate(anim.x, anim.y);
    rotate(angle);
    stroke(c[0], c[1], c[2]);
    strokeWeight(2);
    noFill();
    rect(0, 0, size, size);
    line(-size, 0, size, 0);
    line(0, -size, 0, size);
    noStroke();
    fill(255, 150);
    textSize(20);
    text(anim.type, 10, 10);
  }

  function getColorForSound(s) {
    const colors = {
      'bd':               [150, 50,  200],
      'sd':               [0,   200, 255],
      'hh':               [255, 255, 0  ],
      'gm_epiano1':       [100, 200, 255],
      'gm_acoustic_bass': [150, 50,  200]
    };
    if (colors[s]) return colors[s];
    let cc = s.charCodeAt(0) || 0;
    return [(cc * 123) % 255, (cc * 456) % 255, (cc * 789) % 255];
  }

  function windowResized() { resizeCanvas(windowWidth, windowHeight); }

</script>
</body>
</html>
```

Se le añadió:

- Un objeto global ctrl con los 4 parámetros controlables en tiempo real
- Conexión al Bridge 2 (ws://localhost:8082) para recibir OSC
- Función aplicarControl(param, val, msg) que interpreta las direcciones /ctrl/bombo_size, /ctrl/velocidad, /ctrl/color_hue, /ctrl/glow y /ctrl/xy
- Soporte para el XY pad — un solo mensaje con dos argumentos controla hue y glow simultáneamente
- Función aplicarHue() que rota el tinte de todos los colores en tiempo real
- El parámetro ctrl.bomboSize escalando el tamaño del bombo
- El parámetro ctrl.velocidad multiplicando la duración de todas las animaciones
- El parámetro ctrl.glowAmount reemplazando el valor fijo de background(0, 30)
- Un HUD que muestra el cambio de parametro

La sesión de open stage control:

<img width="213" height="175" alt="openstagecontrol" src="https://github.com/user-attachments/assets/66aef9cc-2f3a-4f83-9465-22a63efb6933" />

Resultados
<img width="1132" height="530" alt="visualesu3" src="https://github.com/user-attachments/assets/6351a61e-479a-4567-bc6a-30fcb090bd41" />
<img width="1103" height="683" alt="visualesu3_2" src="https://github.com/user-attachments/assets/cb12fdcb-65a7-43ed-8b1c-9d0cd5c278e3" />



## Bitácora de reflexión
