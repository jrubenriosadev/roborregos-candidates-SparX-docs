#####################################
Introducción a la resolución de maze
#####################################

Antes de empezar a programar, siempre es bueno definir la solución que se va a implementar, para esto, podemos hacer un uso de **pseudocódigo**.

Antes que nada debemos de conocer los qués antes de los cómos, es decir, debemos de conocer el reto que vamos a resolver, cómo funciona y por su puesto, 
sus implicaciones y restricciones. En este caso, el reto de maze es un laberinto de 5x5 unidades donde **absolutamente todo** puede cambiar de **posición**,
es decir, los obstáculos, paredes, punto de inicio y final del laberinto, además de estar **prohibido** el **premapeo**, es decir, no podemos hacer
un mapa del laberinto antes de prender el robot, por lo que el robot tiene que ser 100% capaz de resolverlo por sí mismo.

Dentro de las implicaciones, el robot tiene que ser capaz de detectar los obstáculos y paredes **siempre**, pero además de eso, tiene que ser capaz de
detectar **colores** en los tiles y **arucos** en las paredes sin interferir con el movimiento del robot. También debe de escanear 4 diferentes colores 
alrededor el mapa, es decir, preferiblemente debe de completar el escaneo antes del bonus final de regresar por el mismo camino.

.. tip::
    Si quieres ver más acerca de cómo funciona el reto, toda la info está en el apartado de retos, ahí podras encontrar todo lo que tienes que saber de
    ambos retos de candidates 2026 :D

Dentro de las restricciones, el robot tiene que ser capaz de ir **recto** para no estrellarse contra las paredes además de tener que girar en su **propio eje**.
Dentro de otras restricciones, el robot no puede escalar las paredes, también de estar prohibidos los **premapeos** y además de **NO** poder hacer uso
de cualquier comunicación directa o indirecta con el robot (aunque si el robot tiene comunicaciones en sí mismo si se puede).


Definir problema
****************

En este caso, hay que considerar que **NO** estamos esperando encontrar el punto más corto de A a B (así me voy a referir al punto inicial y final, donde A es
el inicial y B el final), sino que estamos esperando encontrar una solución donde el robot pueda escanear todos los colores en el piso (4 en total) antes de hacer
el bonus final dentro de una cancha desconocida.

**Hablado de una forma técnica, debemos contruir durante la ejecución un modelo del laberinto que no conocemos explorando sus regiones relevantes, alcanzar el objetivo
y conservar la trayectoria física realizada para poder reproducirla pero al revés.**

Composición del sistema
*************************

El sistema para poder resolver este reto sería algo asi:

1. Localización
2. Mapeo
3. Exploración
4. Navegación
5. Trayectoria

Pero tiene que funcionar esto:

- Sensores
- Reconocedor de obstáculos
- Máquina de estados

No los pongo juntos porque explorar el laberinto y mover el robot **no** son lo mismo, ya que prácticamente son 
problemas diferentes.

.. important::
    Lo **primordial** es que antes de probar **cualquier** algoritmo, el robot debe de ser capaz de **moverse**
    y poder utilizar sus sensores, ya que si no puede hacer eso, nada del algoritmo va a funcionar, por lo que es 
    **fundamental** que hagan pruebas de movimiento y tuneo fino dentro del movimiento.

Localización
*************

Este sistema es literalmente la base de todo el algoritmo.

Hay distintas formas de solucionarlo, pero la verdad es que al no tener una referencia de la posición, debemos
de guardar de alguna manera la información de la posición en la que estamos. Para eso se me ocurre algo como:

.. code-block:: cpp
   :caption: Estructura de posición

   struct Pose {
       int x;
       int y;
       Direction dir;
   };

| Eso para usar algo como:
| x = 2
| y = 3
| dir = NORTH

Eso para representar la tile actual para no depender de coordenadas continuas por asi decirlo.

Posicionamiento vs movimiento
******************************

Ahora, debemos considerar esto que es MUY importante.

El algoritmo actualmente está funcionando con ``Posicion{x,y}`` pero ojo aca, el robot usa cosas como:

1. metros
2. grados
3. velocidad
4. aceleración

Es decir, yo puedo decir que dentro de mi algoritmo me dirija a la posición (2,3) pero en el robot eso sería usar
el navigator para decir algo como girar a EAST, avanzar el tile, detenerse y confirmar la tile.