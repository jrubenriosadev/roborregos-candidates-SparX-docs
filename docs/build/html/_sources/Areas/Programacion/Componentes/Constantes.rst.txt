####################
Constantes y pines
####################

Antes de ver cualquier componente hay que ver **dónde viven los números**. Lo primero que se aprende en un robot es que los valores cambian todo el tiempo: un pin porque
se movió un cable, un PWM porque el piso era más resbaloso, un umbral porque cambió la luz. Si esos números están regados por el código, ajustar el robot
se vuelve una pesadilla.

Por eso todo está en ``include/constants`` en tres archivos:

.. list-table::
    :header-rows: 1
    :widths: 25 75

    * - Archivo
      - Qué contiene
    * - ``pinsConfig.h``
      - Pines de la ESP32, pulsos por vuelta de los encoders, diámetro de la rueda, inversión de los motores.
    * - ``driveConfig.h``
      - Todo lo que tiene que ver con **moverse**: velocidades, ganancias de los PID, tolerancias, umbrales de pared.
    * - ``levelsConfig.h``
      - Constantes del reto Levels: sensor de línea, tiempos de lectura de color, distancias de cada sección.

Por qué ``constexpr`` dentro de un ``namespace``
*************************************************

.. code-block:: cpp
    :caption: Forma de los pines (pinsConfig.h)

    namespace Pins {

        struct Ultrasonic {
            static constexpr uint8_t kLeftEcho = 35;
            static constexpr uint8_t kLeftTrigger = 14;
            static constexpr float maxRange = 15.0f;
        };

    }

Hay dos decisiones aquí que valen la pena entender:

- **constexpr en lugar de #define:** el compilador conoce el tipo del valor y te avisa si lo usas mal, además de que no ocupa memoria en tiempo de ejecución.
- **Agrupar con struct y namespace:** se escribe ``Pins::Ultrasonic::kLeftEcho`` en lugar de ``LEFT_ECHO``. Se lee de dónde viene el valor, y no se pueden mezclar nombres entre sensores.

.. tip::
    La ``k`` al inicio (``kLeftEcho``) es una convención para decir "constante". No hace nada, pero cuando lees el código sabes de un vistazo que ese valor no cambia mientras corre.

Motores: los números que más se ajustan
****************************************

.. code-block:: cpp
    :caption: Motor izquierdo y rueda (pinsConfig.h)

    static constexpr float kLeftPPR = 480.0f;
    static constexpr bool leftInversion = false;

    static constexpr bool rightInversion = true;

    static constexpr float kWheelDiameter = 0.067f;

Estos tres tipos de valor (``PPR``, inversión y diámetro) son los que hacen que el robot sepa **cuánto avanzó** y **hacia dónde**:

- **PPR (pulsos por vuelta):** cuántos pulsos del encoder cuenta el programa en una vuelta completa de la rueda. Con 480 pulsos y una rueda de 6.7 cm, cada pulso equivale a
  más o menos 0.44 mm de avance (:math:`\pi \cdot 0.067 / 480`).
- **Inversión:** los dos motores están montados en espejo, así que para que ambos avancen "hacia adelante" uno de ellos tiene que girar al revés. En lugar de cambiar
  cables, se cambia este ``bool``.
- **Diámetro de la rueda:** convierte las vueltas en metros.

Cómo ajustarlos
================

1. **PPR:** carga el modo de prueba de lazo abierto (:doc:`../Main/CicloPrincipal`), gira la rueda a mano **una vuelta exacta** y lee cuántos pulsos cuenta el encoder.
2. **Diámetro:** pídele al robot que avance una casilla (0.30 m) y **mide con una cinta** cuánto avanzó de verdad. Si avanzó de más, el diámetro real de la rueda es menor al que configuraste (y al revés).
3. **Inversión:** si al mandar "avanzar" una rueda gira hacia atrás, cambia el ``bool`` de ese motor. No muevas cables.

.. important::
    Revisa primero la inversión, luego los pulsos y al final el diámetro. Si la inversión está mal, el encoder cuenta en negativo y nada más tiene sentido.

Un solo lugar para ajustar
***************************

El resto de las constantes (de movimiento, de PID, de pared) se explican en la página donde se usan:

- Ganancias y velocidades en :doc:`../PID/IntroduccionPID`.
- Distancias y paredes en :doc:`../Subsistemas/Introduccion`.
- Colores en :doc:`TCS34725`.
