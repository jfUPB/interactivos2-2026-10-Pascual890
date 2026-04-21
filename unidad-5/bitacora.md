# Unidad 5

## Bitácora de proceso de aprendizaje
Define el concepto de tu obra

### ¿Qué quieres comunicar o provocar con tu obra? (intención artística/estética)

Quiero que sea una obra mas contemplativa con una estética un poco psicodélica.

### ¿En qué contexto se presentará? (sala oscura, espacio abierto, escenario, etc.)

La idea es que se interactue por medio de movimientos detectados en cámara, por lo que debe presentarse en una sala que permita la detección. Una sala oscura sería mejor pero podría presentar problemas de detección de la persona en cámara.

### ¿Cuál es la experiencia que deseas para el público? (contemplativa, participativa, inmersiva, etc.)

Quiero que sea contemplativa e inmersiva, por eso quiero una interacción con cámara para que el usuario pueda involucrarse fácil y rápido a la obra.

### ¿Qué rol tendrá el público? (observador, participante activo, co-creador, etc.)

El público sería participante activo ya que debe haber alguien moviendose para ver los cambios en las visuales.

## Bitácora de aplicación 

``` js
setcps(0.3);

// 1. NOTAS Y MELODÍAS
let notasPad = note("<[c3,eb3,g3] [f3,ab3,c4] [g3,b3,d4] [c3,eb3,g3]>").slow(2);
let notasArp = note("c4 eb4 g4 c5 g4 eb4 d4 b3");
let notasMelodia = note("<c5 ~ eb5 ~> <d5 c5 b4 ~> <g4 ~ c5 ~> <~ d5 eb5 g5>").slow(2);
let notasBajo = note("<c2 f2 g2 c2>").slow(2);

// 2. INSTRUMENTOS Y EFECTOS
let padLayer = notasPad.s("supersaw")
  .lpf(sine.range(200, 1200).slow(8))
  .room(0.8).size(0.8)
  .pan(sine.slow(4))
  .gain(0.4);

let arpLayer = notasArp.s("sawtooth")
  .lpf(800)
  .jux(rev)
  .delay(0.6).delaytime(0.33)
  .room(0.6)
  .gain(0.25);

let melodiaLayer = notasMelodia.s("fm")
  .room(0.9).size(0.9)
  .pan(saw.range(0.2, 0.8).slow(5))
  .gain(0.4);

let bajoLayer = notasBajo.s("square")
  .lpf(250)
  .shape(0.15)
  .gain(0.6);

let percusiones = s("bd(3,8) , ~ hh*2")
  .slow(2)
  .lpf(2000)
  .room(0.4)
  .gain(0.4);

// 3. AUDIO (una sola vez, todo junto)
$: stack(padLayer, arpLayer, melodiaLayer, bajoLayer, percusiones);

// 4. OSC HACIA TDSTRUDELSYNC (cada capa por separado)
$: padLayer.osc();
$: arpLayer.osc();
$: melodiaLayer.osc();
$: bajoLayer.osc();
$: percusiones.osc();
```

## Bitácora de reflexión

Evalúa si el audio generativo que creaste logra la intención estética que planteaste en tu concepto. ¿Qué ajustarías?

El tipo de canción qe quería cambió varias veces en el proceso, pero siento que sí llegué a un resultado que funciona con la intención que quería. Le ajustaría algunas notas de la melodía, algunas no me convencen del todo pero no me se bien los nombres de las notas.

Actualiza el diagrama de sistema de tu obra incorporando los detalles reales de la implementación del audio.



Describe los principales desafíos que enfrentaste y cómo los resolviste.

Hacer música con strudel me parece complicado. Lograr representar exacto lo que uno piensa es muy duro, más que todo lograr la melodía con las notas que uno piensa inicialmente. Finalmente no es tan dificil de resolver buscando referencias de artistas de strudel y haciendo comparaciones con IA, aunque no hay referentes tan variados o de tantos géneros en strudel.



