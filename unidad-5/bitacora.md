# Unidad 5

## Bitácora de proceso de aprendizaje
Define el concepto de tu obra

### ¿Qué quieres comunicar o provocar con tu obra? (intención artística/estética)

Quiero que sea una obra divertida donde el usuario se sienta el centro de la obra y pueda maniobrar con los parametros fácilmente. Con el motivo de juntar el proyecto con la materia de vision artificial, se usarían modelos de tracking del cuerpo. 

### ¿En qué contexto se presentará? (sala oscura, espacio abierto, escenario, etc.)

La idea es que se interactue por medio de movimientos detectados en cámara, por lo que debe presentarse en una sala que permita la detección. Una sala oscura sería mejor pero podría presentar problemas de detección de la persona en cámara, entonces, desde que haya iluminación para que la cámara vea está bien. El fondo no debe ser muy lleno de cosas para facilitar la detección sin interrupciones.

### ¿Cuál es la experiencia que deseas para el público? (contemplativa, participativa, inmersiva, etc.)

Quiero que sea una experiencia participativa, pero en este caso será para una sola persona.

### ¿Qué rol tendrá el público? (observador, participante activo, co-creador, etc.)

El público sería participante activo ya que debe haber alguien moviendose para ver los cambios en las visuales y sonoros.

## Bitácora de aplicación 

``` js
await initAudio()
const leerPuerto = await midin('StrudelMIDI')
const cc = (canal) => leerPuerto(canal)
setcpm(50)

// --- CONTROLES (sin cambios, CCs 10-17) ---
let d_vol = cc(10).range(0, 0.8)
let d_vel = cc(11).range(1, 2).round()
let h_vol = cc(12).range(0, 0.5)
let h_vel = cc(13).range(1, 2).round()
let b_vol   = cc(14).range(0.3, 0.8)
let b_octva = cc(15).range(0, 1).round().mul(-12)
let a_atmos = cc(16).range(0, 0.8)
let a_vel   = cc(17).range(1, 2).round()

// --- KICK: 4 variantes en loop ---
let kick = s("<bd [bd ~] [bd ~] [bd bd:2]>").bank("Linn9000").gain(d_vol).fast(d_vel)

// --- SNARE: ghost notes con volumen variable ---
let snare = s("<[~ sd] [~ sd:2] [~ sd] [~ sd ~ sd:0]>").bank("Linn9000").gain(d_vol.mul(0.9)).fast(d_vel)

// --- HATS: abiertos y cerrados con velocidades variables ---
let hats = s("<[hh hh hh hh] [hh hh oh hh] [hh hh hh oh] [hh oh hh hh]>").bank("Linn9000").gain(h_vol).fast(h_vel)

// --- BAJO: 8 compases en G menor, con notas de paso ---
let bajo = note("<[g2 ~ d2 ~] [c2 ~ g2 ~] [e2 ~ b2 ~] [d2 ~ a2 ~] [g2 ~ f2 ~] [c2 ~ e2 ~] [d2 ~ c2 d2] [g2 ~ ~ ~]>").sound("piano").transpose(b_octva).gain(b_vol).room(0.6).lpf(900).legato(0.85)

// --- MELODÍA: 8 frases con contorno melódico real ---
let arpegio = note("<[g4 b4 d5 ~] [c5 e5 g5 e5] [d5 ~ b4 g4] [f4 a4 c5 ~] [g4 d5 ~ b4] [e5 d5 b4 g4] [c5 ~ g4 a4] [b4 d5 g5 ~]>").sound("square").lpf(900).gain(0.18).delay(a_atmos).delayfeedback(a_atmos.mul(0.75)).delaytime(0.375).room(0.55).fast(a_vel).jux(rev)

// --- EJECUCIÓN ---
$: stack(kick, kick.osc())
$: stack(snare, snare.osc())
$: stack(hats, hats.osc())
$: stack(bajo, bajo.osc())
$: stack(arpegio, arpegio.osc())
```

## Bitácora de reflexión

Evalúa si el audio generativo que creaste logra la intención estética que planteaste en tu concepto. ¿Qué ajustarías?

El tipo de canción qe quería cambió varias veces en el proceso, pero siento que sí llegué a un resultado que funciona con la intención que quería. Le ajustaría algunas notas de la melodía, algunas no me convencen del todo pero no me se bien los nombres de las notas. Fue un proceso de intentar muchas varaciones hasta tratar de llegar a algo que al menos fuera coherente, másque todo porque cambiará durante el uso y no puede quedar muy loco.

Hacer música con strudel me parece complicado. Lograr representar exacto lo que uno piensa es muy duro, más que todo lograr la melodía con las notas que uno piensa inicialmente. Finalmente no es tan dificil de resolver buscando referencias de artistas de strudel y haciendo comparaciones con IA, aunque no hay referentes tan variados o de tantos géneros en strudel. Como la obra puede ser manipulada en tiempo real de forma rápida, lo más duro fue logras las conexiones midi y del bridge necesarias, y lograr un patrón que no sonara muy desfasado, siento que hay mucho por mejorar en el aspecto musical.



