#####################
Área de Programación
#####################

Dentro de la programación de RIB-EYE se utilizaron distintas herramientas para la resolución de los retos.
Las siguientes explicaciones de código son asumiendo que ya se tienen ciertos conocimientos previos de C++ y programación de
microcontroladores como Arduino.

Herramientas utilizadas
************************

1. **PlatformIO:** Este es un entorno de desarrollo integrado (IDE) que te permite escribir, compilar y cargar código en prácticamente cualquier
microcontrolador, ya que PlatformIO es compatible con muchísimos microcontroladores y placas. En nuestro caso, utilizamos un microcontrolador ESP32.

2. **C++:** Este es un lenguaje de programación que tiene un fuerte enfoque en el control de bajo nivel, lo que nos permite tener una manipulación
más directa del hardware y de los recursos del microcontrolador, además de ser un lenguaje bastante eficiente en cuanto a su velocidad y tiempo de ejecución.

Retos de cada semana
*********************

- **Semana 2:** El mayor reto fue adaptarme a cpp luego de haber trabajado por mucho tiempo en Java, pero las estructuras para los códigos ya las tenía bastante ensayadas, por lo que no fue difícil darle estructura al código

- **Semana 3:** Lo más difícil fue buscar documentación de las configuraciones de sensores, aprendí que es más fácil buscarlas en Arduino y simplemente hacer la conversión a ESP32

- **Semana 4:** Lo más díficl de esta semana fue empezar con algunas pruebas, ya que teníamos que tener bastantes iteraciones dentro de las pruebas.

- **Semana 5:** En la semana 5 no hubieron muchas complicaciones, pero el hecho de tener que recalibrar cada que haya cierto cambio si fue algo tedioso, pero me volvi más consistente en eso.

- **Semana 6:** Lo más difícil fue hacer los algoritmos para ambas competencias en pseudocódigo al no tener el robot en físico, pero una vez que el motion funciona, el teórico tiene lógica de que también.

 
Cómo está pensado el código
****************************

Antes de entrar a cada archivo, conviene entender **la idea completa**, porque casi todo el código sigue la misma lógica: construir de abajo hacia arriba,
donde cada capa **solo conoce a la de abajo** y resuelve **un solo problema**.

.. code-block:: text
    :caption: Capas del código

    Retos (maze, levels) dicen los qués
    Máquina de estados deciden cuando
    Subsistemas (Drive) saben como mover
    Contenedor (RobotContainer) saben que tiene el robot
    Componentes (Motor, Bno...) hablan con el hardware

Cada capa tiene su propia página:

1. **Componentes:** una clase por cada sensor o actuador. Esconden los detalles del hardware (I2C, pulsos, interrupciones).
2. **Contenedor:** el dueño de todos los sensores. Los inicializa, los actualiza y le da al resto del código lecturas ya filtradas y con sentido ("hay pared al frente").
3. **PID:** el controlador que nos permite ir recto y girar con precisión.
4. **Subsistemas:** el ``Drive`` junta los motores y los PID para ofrecer movimientos completos como "avanza una casilla".
5. **Máquina de estados:** ordena los movimientos en el tiempo sin bloquear nunca el programa.

.. tip::
    Si estás empezando, léelas en ese orden. Cada página explica qué problema resuelve y por qué se resolvió así, antes de mostrar el poco código
    que realmente importa.

.. important::
    En estas páginas **no vamos a mostrar todo el código**, solo las partes de configuración y los fragmentos clave. El código completo está en el repositorio del robot.


Resolución de retos
********************

.. toctree::
   :caption: Ciclo principal
   :maxdepth: 2

   Ciclo principal y modos de prueba <Main/CicloPrincipal>

.. toctree::
   :caption: Componentes
   :maxdepth: 2

   Configuración de componentes <ComponentesProgra>

.. toctree::
   :caption: PID
   :maxdepth: 2

   Introducción <PID/IntroduccionPID>

.. toctree::
   :caption: Contenedor y Máquina de Estados
   :maxdepth: 2

   Introducción al contenedor (RobotContainer) <Contenedor/Introduccion>
   Introducción a la máquina de estados (StateMachine) <MaquinaEstados/Introduccion>

.. toctree::
   :caption: Subsistemas
   :maxdepth: 2

   Introducción a los subsistemas <Subsistemas/Introduccion>


.. toctree::
   :caption: Algoritmo de resolución de maze
   :maxdepth: 1

   Planteamiento de resolución de maze<Algoritmo/Planteamiento>
