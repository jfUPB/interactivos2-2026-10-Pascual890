# Unidad 2

## Bitácora de proceso de aprendizaje
<img width="1258" height="650" alt="sfi2 u2" src="https://github.com/user-attachments/assets/829454e8-b178-4843-9076-85aa904fedaa" />


## Bitácora de aplicación 
Primero tuve que modificar el código de strudel.

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
De este modo, ya se pueden mandar señales usando el mismo beat de strudel que tenía desde el ejercicio pasado.

Luego se modificó el archivo de ejemplo de visuales

Se realizaron **3 modificaciones principales** al código base del profesor para mejorar la experiencia visual y sincronizar mejor con los parámetros musicales del código Strudel.

---

## MODIFICACIÓN 1: Efecto de Rebote en el Bombo (bd)

### Código Original:
```javascript
function dibujarBombo(p, c) {
  let d = lerp(100, 600, p);
  let alpha = lerp(255, 0, p);
  fill(c[0], c[1], c[2], alpha);
  circle(width / 2, height / 2, d);
}
```

### Código Modificado:
```javascript
function dibujarBombo(p, c) {
  let bounce = sin(p * PI);
  let d = lerp(80, 400, bounce);
  let alpha = lerp(255, 0, p);
  fill(c[0], c[1], c[2], alpha);
  circle(width / 2, height / 2, d);
}
```

### ¿Qué cambia?
- **Antes**: El círculo crecía linealmente según el progreso (`p`)
- **Ahora**: El círculo tiene un efecto de "rebote" usando la función `sin(p * PI)`

### ¿Por qué?
La función seno crea una curva suave que:
1. Empieza en 0
2. Crece hasta 1 a la mitad del tiempo
3. Decrece de vuelta a 0

Esto simula el comportamiento físico de un bombo real que golpea y rebota.

---

## MODIFICACIÓN 2: Hi-Hats con Estela de Luz Reactiva a Velocity

### Código Original:
```javascript
function dibujarHat(anim, p, c) {
  let sz = lerp(40, 0, p);
  fill(c[0], c[1], c[2]);
  rect(anim.x, anim.y, sz, sz);
}
```

### Código Modificado:
```javascript
function dibujarHat(anim, p, c) {
  // La altura de la estela depende de la velocity
  let maxHeight = map(anim.velocity, 0.2, 0.9, 100, 300);
  let currentHeight = lerp(0, maxHeight, p);
  
  // Posición Y sube con el tiempo
  let yPos = anim.y - currentHeight;
  
  // La estela se desvanece mientras sube
  let alpha = lerp(255, 0, p);
  let thickness = lerp(8, 2, p);
  
  // Dibujamos la línea vertical
  stroke(c[0], c[1], c[2], alpha);
  strokeWeight(thickness);
  line(anim.x, anim.y, anim.x, yPos);
  
  // Punto brillante en la punta
  noStroke();
  fill(255, 255, 255, alpha);
  circle(anim.x, yPos, thickness * 2);
}
```

### ¿Qué cambia?
- **Antes**: Cuadrado amarillo que se reducía en su posición
- **Ahora**: Estela de luz vertical que sube y responde a velocity

### ¿Por qué?
En el código Strudel tenemos:
```javascript
s("hh*4").gain(0.7).velocity(sine.range(0.2, 0.9).slow(1))
```

La velocity varía entre 0.2 y 0.9 siguiendo una onda seno. La visualización ahora refleja esto:
- **Velocity baja (0.2)**: Estela corta (100px)
- **Velocity alta (0.9)**: Estela larga (300px)

### Elementos visuales:
1. **Línea vertical**: Representa la estela que sube
2. **Grosor decreciente**: De 8px a 2px para efecto de disipación
3. **Punto brillante**: Círculo blanco en la punta para marcar el extremo
4. **Alpha decreciente**: Se desvanece mientras sube

---

## MODIFICACIÓN 3: Piano con Ondas Concéntricas

### Código Original:
No existía visualización específica para el piano (usaba `dibujarDefault`)

### Código Nuevo:
```javascript
function dibujarPiano(p, c) {
  noFill();
  stroke(c[0], c[1], c[2], lerp(200, 0, p));
  strokeWeight(3);
  let d = lerp(50, 300, p);
  circle(width / 2, height / 2, d);
}
```

### ¿Qué hace?
Crea un círculo que se expande desde el centro con:
- Trazo azul claro [100, 200, 255]
- Expansión de 50px a 300px
- Desvanecimiento gradual

Código final de visuales

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

  function setup() {
    createCanvas(windowWidth, windowHeight);
    rectMode(CENTER);
    noStroke();

    const socket = new WebSocket('ws://localhost:8081');

    socket.onmessage = (event) => {
        const msg = JSON.parse(event.data);

        console.log('Mensaje OSC recibido:', msg);
        let params = {};
        
        for (let i = 0; i < msg.args.length; i += 2) {
            params[msg.args[i]] = msg.args[i+1];
        }
        eventQueue.push({ 
          timestamp: msg.timestamp, 
          sound: params.s,
          delta: params.delta || 0.25,
          velocity: params.velocity || 0.7, // guardamos velocity
          params: params });

        eventQueue.sort((a, b) => a.timestamp - b.timestamp);
    };
  }

  function draw() {
    background(0, 30); 

    let now = Date.now() + LATENCY_CORRECTION;

    while (eventQueue.length > 0 && now >= eventQueue[0].timestamp) {
        let ev = eventQueue.shift();

        activeAnimations.push({
        startTime: ev.timestamp,
        duration: ev.delta * 1000,
        type: ev.sound,
        velocity: ev.velocity, // pasamos velocity a la animación
        x: random(width * 0.2, width * 0.8), 
        y: random(height * 0.2, height * 0.8),
        color: getColorForSound(ev.sound)
        })        

    }
    
    for (let i = activeAnimations.length - 1; i >= 0; i--) {
      let anim = activeAnimations[i];
      
      let elapsed = now - anim.startTime;
      let progress = elapsed / anim.duration;

      if (progress <= 1.0) {
        dibujarElemento(anim, progress);
      } else {
        activeAnimations.splice(i, 1);
      }
    }
  }

  function dibujarElemento(anim, p) {
    push();
    const color = anim.color;
    
    switch (anim.type) {
      case 'bd':
        dibujarBombo(p, color);
        break;

      case 'sd':
        dibujarCaja(p, color);
        break;

      case 'hh':
        dibujarHat(anim, p, color);
        break;

      case 'gm_epiano1':
        dibujarPiano(p, color);
        break;

      default:
        dibujarDefault(anim, p, color);
        break;
    }
    pop();
  }


  // MODIFICACIÓN 1: Bombo con efecto de rebote
  function dibujarBombo(p, c) {
    // Usamos sin() para crear un efecto de rebote más natural
    let bounce = sin(p * PI);
    // Cambiamos el rango de tamaño: antes era (100, 600), ahora más pequeño
    let d = lerp(80, 400, bounce);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    circle(width / 2, height / 2, d);
  }

  function dibujarCaja(p, c) {
    let w = lerp(width, 0, p);
    let alpha = lerp(255, 0, p);
    fill(c[0], c[1], c[2], alpha);
    rect(width / 2, height / 2, w, 50);
  }

  // MODIFICACIÓN 2: Hi-hat con estela de luz que sube
  function dibujarHat(anim, p, c) {
    // La altura de la estela depende de la velocity
    let maxHeight = map(anim.velocity, 0.2, 0.9, 100, 300);
    let currentHeight = lerp(0, maxHeight, p);
    
    // Posición Y sube con el tiempo
    let yPos = anim.y - currentHeight;
    
    // La estela se desvanece mientras sube
    let alpha = lerp(255, 0, p);
    let thickness = lerp(8, 2, p);
    
    // Dibujamos la línea vertical (estela)
    stroke(c[0], c[1], c[2], alpha);
    strokeWeight(thickness);
    line(anim.x, anim.y, anim.x, yPos);
    
    // Punto brillante en la punta
    noStroke();
    fill(255, 255, 255, alpha);
    circle(anim.x, yPos, thickness * 2);
  }

  // MODIFICACIÓN 3: Piano con ondas concéntricas simples
  function dibujarPiano(p, c) {
    noFill();
    stroke(c[0], c[1], c[2], lerp(200, 0, p));
    strokeWeight(3);
    let d = lerp(50, 300, p);
    circle(width / 2, height / 2, d);
  }

  function dibujarDefault(anim, p, c) {
    let size = lerp(100, 0, p);
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
      'bd': [150, 50, 200],        // color bombo
      'sd': [0, 200, 255],
      'hh': [255, 255, 0],
      'gm_epiano1': [100, 200, 255],
      'gm_acoustic_bass': [150, 50, 200]
    };

    if (colors[s]) return colors[s];

    let charCode = s.charCodeAt(0) || 0;
    let r = (charCode * 123) % 255;
    let g = (charCode * 456) % 255;
    let b = (charCode * 789) % 255;
    return [r, g, b];
  }  

  function windowResized() { resizeCanvas(windowWidth, windowHeight); }


</script>
</body>

</html>
```
## Bitácora de reflexión

