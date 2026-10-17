#################################
Máquina de estados (StateMachine)
#################################

Un robot tiene que hacer cosas **en orden**: avanzar una casilla, esperar a estabilizarse, mirar si hay pared, decidir, girar, volver a avanzar. La forma más natural de escribir eso es una lista de instrucciones:

.. code-block:: text
    :caption: Cómo NO se hace (pseudocódigo)

    avanzar(30 cm);
    delay(800);
    si hay pared al frente: girar(90);
    avanzar(30 cm);

Esto **no funciona en un robot**. Mientras el robot "avanza 30 cm" (o espera con un ``delay``), el programa se queda atorado en esa línea: no lee sensores, no corre el PID, no reacciona a nada (ver :doc:`../Main/CicloPrincipal`).
Un robot necesita que **cada vuelta del ciclo sea cortísima** y regrese enseguida.

Qué es una máquina de estados
******************************

La solución es dividir la tarea en **estados**. En cada ciclo, el robot **solo hace lo que le toca en el estado actual**, y si ya terminó, cambia de estado. Es como un mapa de "dónde estoy" y "a dónde voy después".

.. code-block:: cpp
    :caption: Los estados (StateMachine.hpp)

    enum class States {
        WAITING,
        PREP,
        MOVING,
        DELAY,
        DECIDE,
        APPROACH,
        STEERING,
        TURN_DELAY,
        STOPPED
    };

El ``update()`` es un ``switch`` sobre ese estado. Cada ``case`` hace **un pedacito de trabajo y regresa**:

.. code-block:: cpp
    :caption: Estructura de update() (StateMachine.cpp)

    switch (_currentState) {
        case States::WAITING: {
            _container.getDrive().stop();
            if(!_container.tofReady()) break;

Las ventajas son varias: el robot **nunca se bloquea**, siempre se sabe **en qué paso está** (útil para depurar: se imprime el estado) y agregar un comportamiento nuevo significa agregar un estado, no reescribir todo.

Qué hace nuestra máquina de estados
************************************

Esta máquina resuelve el reto casilla por casilla, con una regla simple: **avanzar mientras no haya pared, y si hay pared, decidir hacia dónde girar**.

.. list-table::
    :header-rows: 1
    :widths: 18 82

    * - Estado
      - Qué hace
    * - ``WAITING``
      - El robot está **detenido** hasta que el sensor de color ve **amarillo** (la casilla de inicio). Entonces guarda su rumbo actual.
    * - ``PREP``
      - Prepara el movimiento de **una casilla** (pone el contador de distancia en cero y fija el rumbo objetivo).
    * - ``MOVING``
      - Avanza la casilla con el PID. Al terminar se detiene.
    * - ``DELAY``
      - Espera un momento detenido (800 ms) para que el robot se asiente y los sensores entreguen lecturas frescas.
    * - ``DECIDE``
      - Con el robot quieto, revisa las paredes y decide: seguir recto o girar (ver más abajo).
    * - ``APPROACH``
      - Si va a girar y hay una pared enfrente dentro de la casilla, se acerca a ella para centrarse (si no hay pared cerca, el robot gira directo).
    * - ``STEERING``
      - Gira hasta el ángulo objetivo.
    * - ``TURN_DELAY``
      - Una pausa corta (300 ms) después del giro, antes de avanzar.
    * - ``STOPPED``
      - Terminó la ruta. Espera a ver **verde** para reanudar.

.. code-block:: text
    :caption: Cómo fluyen los estados

    WAITING --(amarillo)--> PREP --> MOVING --> DELAY --> DECIDE
                             ^                              |
                             |         sin pared al frente  |
                             +------------------------------+
                             |
                             |  con pared: gira
                             |     DECIDE --> APPROACH --> STEERING --> TURN_DELAY --+
                             +-------------------------------------------------------+

    (si se detecta cian en la casilla, MOVING pasa a STOPPED; con verde, STOPPED reanuda en PREP)

Por qué hay pausas (DELAY y TURN_DELAY)
========================================

Parecen tiempo perdido, pero tienen un motivo: **cuando el robot se detiene, todavía se está moviendo un poquito** (inercia) y los sensores necesitan tiempo para dar una lectura nueva (el ToF tarda ~33 ms y cada
ultrasónico se lee cada 80 ms). Decidir con lecturas viejas o con el robot meciéndose lleva a errores. Los tiempos son constantes de ``driveConfig.h`` (``kDecideDelayMs``, ``kSteerSms``), así que se pueden ajustar sin tocar el código.

La decisión
============

.. code-block:: cpp
    :caption: Regla de decisión (StateMachine.cpp)

    if (!_container.frontWall()) {
        _currentState = States::PREP;
        break;
    }

    const bool leftOpen = !_container.leftWall();
    const bool rightOpen = !_container.rightWall();

    if (leftOpen && rightOpen) {
        startTurn(Cfg::kRightPref ? Cfg::kSteerDeg90 : -Cfg::kSteerDeg90);
    } else if (rightOpen) {
        startTurn(Cfg::kSteerDeg90);
    } else if (leftOpen) {
        startTurn(-Cfg::kSteerDeg90);
    } else if (Cfg::kDeadEnd180) {
        startTurn(2.0f * Cfg::kSteerDeg90);
    } else {
        _currentState = States::STOPPED;
    }

Se lee casi como una oración: **si no hay pared al frente, sigue. Si hay pared y ambos lados están abiertos, prefiere un lado. Si solo uno está abierto, ve hacia él. Si ninguno, es un callejón: da la vuelta.**
Fíjate que ``DECIDE`` no mueve nada, solo **elige el siguiente estado**. El movimiento lo hace ``Drive`` (ver :doc:`../Subsistemas/Introduccion`) y la lectura de paredes viene del contenedor (ver :doc:`../Contenedor/Introduccion`).

Las preferencias (``kRightPref``, ``kDeadEnd180``) están en ``driveConfig.h`` como constantes. Así, **cambiar el comportamiento del robot es cambiar un bool**, no reescribir la lógica.

El rumbo "bloqueado"
=====================

.. code-block:: cpp
    :caption: Acumular el objetivo en lugar de leerlo (StateMachine.cpp)

    void StateMachine::startTurn(float deltaDeg) {
        _lockedYaw = normalizeY(_lockedYaw + deltaDeg);

Esta es una decisión pequeña pero importante. Al girar 90° a la derecha, **no se lee el yaw actual y se le suma 90°**: se suma 90° al **rumbo objetivo anterior** (``_lockedYaw``). Si el robot se quedó
1.5° corto en un giro, ese error **no se arrastra** al siguiente: el siguiente objetivo se calcula desde el rumbo que debía tener, no del que terminó teniendo. Así, después de muchos giros, el robot sigue alineado con la cuadrícula.

.. important::
    El rumbo objetivo se fija **una vez** (cuando el robot ve amarillo) y todos los demás rumbos se calculan a partir de él. Es la misma razón por la que existe ``resetHeading()`` en el BNO055 (ver :doc:`../Componentes/BNO055`).

El color como señal
====================

En esta máquina el color no es solo información, es una **señal de control**: **amarillo** arranca, **cian** detiene al robot al terminar la casilla y **verde** reanuda desde ``STOPPED``. Es la forma más sencilla de que el
robot sepa "empezó" y "terminó" sin necesitar ningún botón.

Cuando el reto se vuelve más grande: el maze
*********************************************

Con esta máquina de estados el robot **sigue paredes**, pero no **recuerda** nada. El reto de maze requiere explorar todo el laberinto, guardar el mapa y regresar por el mismo camino (ver :doc:`../Algoritmo/Planteamiento`).
Eso es demasiado para un solo ``switch``, así que se dividió en piezas, cada una con **un solo trabajo**:

.. list-table::
    :header-rows: 1
    :widths: 25 75

    * - Pieza
      - Trabajo
    * - ``Maze``
      - Guarda el mapa: qué casillas conoce, dónde hay paredes, qué color tiene cada una.
    * - ``Explorer``
      - Decide **a qué casilla ir** para explorar (prefiere frente, izquierda, derecha y atrás, en ese orden) y regresa sobre sus pasos cuando no hay nada nuevo.
    * - ``Navigator``
      - Lleva la **posición y dirección** en la cuadrícula, y traduce "ve al norte" en "gira y avanza una casilla".
    * - ``PathPlanner``
      - Calcula el camino más corto entre dos casillas con lo que ya conoce del mapa.
    * - ``Trajectory``
      - Guarda cada movimiento exitoso, para poder **regresar por el mismo camino** al final.
    * - ``MazeStateMachine``
      - El coordinador: decide **cuándo** usar cada pieza.

Su máquina de estados principal es más corta de lo que parece:

.. code-block:: cpp
    :caption: Los estados del maze (MazeStateMachine.hpp)

    enum class MazeState { INIT, EXPLORING, GOING_TO_FINISH, RETURNING, DONE, FAULT };

**Explora** hasta no tener nada nuevo que ver, **va a la meta** por el camino más corto, **regresa** por el mismo camino y termina. Y si algo sale mal (un sensor no entrega nada, un movimiento falla, se pierde la posición),
**no se queda tonto ni sigue a ciegas**: pasa a ``FAULT`` con un **error con nombre** (``MazeError``) que se imprime y dice qué falló.

Interfaces: el algoritmo no conoce al robot
============================================

La decisión de diseño más importante del maze es esta:

.. code-block:: cpp
    :caption: Lo único que el algoritmo sabe del hardware (Sensors.hpp y MotionController.hpp)

    class Sensors{
        public:
            virtual ~Sensors() {}
            virtual bool ready() = 0;
            virtual void beginScan() = 0;
            virtual bool pollScan(SensorReadings& out) = 0;

    class MotionController {
        public:
            virtual ~MotionController() {}
            virtual bool begin() = 0;
            virtual void startTurn(Direction target) = 0;
            virtual void startMove(Direction heading) = 0;
            virtual NavStatus update() = 0;
            virtual void abort() = 0;
            // ...

El algoritmo **solo sabe pedir** cosas como "escanea la casilla" o "gira hacia el este" y **solo sabe recibir** respuestas como "hay pared al frente" o "terminé". No sabe qué es un motor, un BNO055 o un PID.
Quien traduce esas peticiones al robot real son dos clases pequeñas (``RobotSensors`` y ``DriveMotion``) que usan el contenedor y el ``Drive``.

Esto trae tres ventajas muy prácticas:

1. **Se puede probar sin robot.** Como el algoritmo solo habla con esas dos interfaces, se podría correr contra sensores y movimiento simulados, y si algo sale mal sabrías que **es el algoritmo**, no el hardware.
2. **Se puede cambiar el hardware sin tocar el algoritmo.** Es la misma separación entre *posicionamiento* y *movimiento* de la que habla :doc:`../Algoritmo/Planteamiento`.
3. **Se reusa.** El reto Levels usa las mismas piezas (``Maze``, ``Navigator``, ``DriveMotion``) y solo agrega las suyas.

Escanear sin creerle a una sola lectura
========================================

Para saber qué hay en una casilla, ``RobotSensors`` no pregunta una vez: **toma muchas lecturas durante una ventana de tiempo y vota**.

.. code-block:: cpp
    :caption: Escaneo por votación (RobotSensors.cpp)

    if (el < maze::MazeConfig::kScanSettleMs) return false;
    if (el < maze::MazeConfig::kScanSettleMs + maze::MazeConfig::kScanWindowMs) { sample(); return false; }

    out.fWall = front_ * 2 > n_;
    out.lWall = left_ * 2 > n_;
    out.rWall = right_ * 2 > n_;

Primero **espera** (``kScanSettleMs``, 700 ms) a que el robot se asiente y luego **muestrea** durante ``kScanWindowMs`` (400 ms). Al final, hay pared si **más de la mitad** de las muestras dijeron que había.
El color se decide igual, pero exigiendo que el ganador tenga al menos 60% de los votos (``kColorAgreePct``). Si no hay acuerdo suficiente, la lectura se marca como **inválida** y la máquina de estados decide si repite el escaneo o falla.

.. tip::
    Los tiempos y porcentajes están en ``MazeConfig.hpp``. Si el robot se equivoca de pared o de color con frecuencia, **alarga la ventana o sube el porcentaje de acuerdo**; si es muy lento, hazlo al revés.
