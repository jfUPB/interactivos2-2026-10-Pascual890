# Unidad 1

## Bitácora de proceso de aprendizaje
<img width="782" height="538" alt="sfi2 actividad1" src="https://github.com/user-attachments/assets/1ffa574d-3693-41eb-9f53-9d0794c3cb02" />
<img width="2555" height="1524" alt="sfi2act1" src="https://github.com/user-attachments/assets/cf6acb39-aa0f-4f32-9f96-c51c07980336" />
<img width="971" height="314" alt="act1" src="https://github.com/user-attachments/assets/4db5167a-fe58-46fc-8d6a-93966808851a" />


## Bitácora de aplicación 
Para la aplicación quería hacer un beat lento. Quería hacer drums, melodia y bajo.



Empecé por los drums
Se definió un setcps(0.75) para obtener un ritmo que que no fuera muy rápido
En lugar de un ritmo lineal, se utilizaron silencios (~) y agrupaciones [ ]. El bombo (bd) se programó con una variación de golpe doble al final de cada 4 ciclos usando <bd!3 [bd*2]> para que no suene muy plano.
Se utilizó hh*8 para mantener una subdivisión que se puede cambiar.
Se usó .gain en todo para controlar el nivel de cada instrumento.
Asi estaban inicialmente. 
```js
stack(
  // Drums
  s("bd [~ bd] <bd!3 [bd*2]> ~").gain(1.2),
  s("~ sd").gain(0.8),
  s("hh*8").gain(0.7), 
```

Luego busqué como se podia cambiar el volumen en diferentes compases para los hihat.

```js
stack(
  // Drums
  s("bd [~ bd] <bd!3 [bd*2]> ~").gain(1.2),
  s("~ sd").gain(0.8),
  s("hh*8").gain(0.7).velocity(sine.range(0.2, 0.8).slow(4)), 
```
.velocity sirve para cambiar la fuerza con que toca, luego el sine.range define una onda entre dos valores y slow define cada cuanto sucede el cambio de volumen entre los 2 valores.
saw, square, tri.

Para hacer las notas de la melodia y el bajo, se usaron unos let con notas aleatorias hasta que sonara bien.

```js
let base = "<d2 g1>/2" 
let notas = "<[f3,a3,c4] [bb3,d3,f4]>/2" 
```
con /2 para que dure 2 compases

Luego en el stack 
```js
// Bajo
  note(base).s("gm_acoustic_bass")
    .lpf(800)
    .gain(1),

  // Piano
  note(stack(base, notas)).s("gm_epiano1")
    .gain(0.2)
    .room(3) 
    .lpf(sine.range(500, 1500).slow(8)) 
```
Ambos tienen filtros low pass para filtrar sonidos agudos y jugar con los sonidos. el piano tambien usa un sine para que cambie el sonido en diferentes momentos del beat.
Para crear un efecto extra puse el .room que es reverb.

Final
```js
setcps(0.75)

let base = "<d2 g1>/2" 
let notas = "<[f3,a3,c4] [bb3,d3,f4]>/2" 

stack(
  // Drums
  s("bd [~ bd] <bd!3 [bd*2]> ~").gain(1.2),
  s("~ sd").gain(0.8),
  s("hh*8").gain(0.7).velocity(sine.range(0.4, 0.8).slow(1)), 

  // Bajo
  note(base).s("gm_acoustic_bass")
    .lpf(800)
    .gain(1),

  // Piano
  note(stack(base, notas)).s("gm_epiano1")
    .gain(0.2)
    .room(3)
    .lpf(sine.range(500, 1500).slow(8)) 
)
```


## Bitácora de reflexión
<img width="865" height="300" alt="sfi2 act5" src="https://github.com/user-attachments/assets/05d01e5c-4079-424a-a0c0-9c75eb9f5569" />

Primero, al ejecutar el código en Strudel, la función .osc() convierte cada sonido en un mensaje digital. 
Segundo, el OSCBridge actúa como un intermediario que recibe esos datos y los transmite en tiempo real hacia el navegador. 
Por último, el archivo visualesHouse.html recibe la información y utiliza p5.js para transformar esos mensajes en cambios visuales, logrando que la animación reaccione exactamente al ritmo de la música.




