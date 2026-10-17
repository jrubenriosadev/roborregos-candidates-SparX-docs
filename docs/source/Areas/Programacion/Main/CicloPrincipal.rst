###################################
Ciclo principal y modos de prueba
###################################

Esta es la primera página que conviene leer, porque explica **cómo corre todo**: en qué momento se llama a cada cosa y cómo probamos el robot sin tener que escribir un programa nuevo cada vez.
Todo vive en ``main.cpp``.

El ciclo de control de 20 ms
*****************************

Un robot que se mueve con PID necesita algo muy importante: **que el controlador se ejecute siempre a la misma frecuencia**. Si una vuelta del ``loop()`` tarda 5 ms y la siguiente 40 ms, el cálculo
de la derivada y de la integral del PID se descompone (ver :doc:`../PID/IntroduccionPID`).

Por eso ``loop()`` no corre "lo más rápido que pueda", sino **una vez cada 20 ms** (50 veces por segundo):

.. code-block:: cpp
    :caption: Periodo fijo (main.cpp)

    void loop() {
        const unsigned long now = micros();
        if ((long)(now - nextTick) < 0) return;
        nextTick += kControlPeriod;
        if ((long)(now - nextTick) > (long)kControlPeriod) nextTick = now + kControlPeriod;

        container.update();
        testLoop();

Esto se entiende por partes:

1. **Esperar su turno.** Si todavía no toca la siguiente vuelta, no hace nada y sale.
2. **Programar la siguiente vuelta.** Con ``nextTick += kControlPeriod`` se programa la siguiente vuelta **sumando** el periodo al tiempo anterior (no a "ahora"). Así no se acumula error: si una vuelta salió con 1 ms de retraso, la siguiente lo compensa.
3. **No ponerse al corriente.** Si el programa se atrasó **más de un periodo completo** (por ejemplo, por una lectura lenta), no intenta "ponerse al corriente" con varias vueltas seguidas; mejor se reinicia el reloj.
4. **El orden importa:** primero ``container.update()`` (leer todos los sensores) y **después** la lógica del robot, para que siempre decida con lecturas frescas.

.. tip::
    Toda la lógica del robot (máquina de estados, PID, retos) debe ser **no bloqueante**: cada vuelta hace un poquito de trabajo y regresa. Un ``delay()`` largo en cualquier parte congela
    sensores y PID por igual.

Dos buses I2C
**************

.. code-block:: cpp
    :caption: Configuración de buses (main.cpp)

    Wire.begin(21, 22);
    Wire.setClock(100000);
    Wire.setTimeOut(10);

    Wire1.begin(5, 18);

- ``Wire`` (pines 21 y 22) para el BNO055 y el TCS34725, y ``Wire1`` (pines 5 y 18) para el VL53L0X. La razón de usar dos buses está en :doc:`../Componentes/VL53L0X`.
- ``setClock(100000)``: el bus a 100 kHz, la velocidad estándar. Es más lenta que la rápida (400 kHz), pero **más tolerante a cables largos y ruido**, que en un robot con motores sobran.
- ``setTimeOut(10)``: si el bus se traba, la lectura se rinde a los 10 ms en lugar de quedarse colgada.

Recuperar un bus trabado
=========================

.. code-block:: cpp
    :caption: Liberar el bus (main.cpp)

    for (int i = 0; i < 16; i++) {
        digitalWrite(sclPin, LOW);
        delayMicroseconds(10);
        digitalWrite(sclPin, HIGH);
        delayMicroseconds(10);
        if (digitalRead(sdaPin) == HIGH) break; // SDA liberado
    }

Es normal que, si la ESP32 se reinicia **justo a la mitad de una comunicación**, un sensor se quede esperando y mantenga la línea de datos (SDA) en bajo, y entonces nadie vuelve a hablar por ese bus.
La solución es mandar **hasta 16 pulsos de reloj** hasta que el sensor termine lo que estaba haciendo y suelte la línea. ``recoverI2C()`` se llama **antes** de iniciar los buses.

Modos de prueba
****************

El robot es un sistema donde todo depende de todo: si el robot no gira bien, no sabes si es el sensor, el PID o la máquina de estados. Por eso ``main.cpp`` tiene un **modo de prueba** que se elige
con una sola línea y que prende **solo una parte del robot** a la vez:

.. code-block:: cpp
    :caption: Selector de modo (main.cpp)

    // 1=color  2=openloop  3=tile PID  4=statemachine  5=MAZE  6=sensores
    // 7=LEVELS fase 1 (LIME->RED)  8=LEVELS fase 2 (RED->RED)  9=LEVELS fase 3 (RED->LIME)  10=LEVELS completo
    #define TEST_MODE 4

.. list-table::
    :header-rows: 1
    :widths: 12 25 63

    * - Modo
      - Qué prueba
      - Para qué sirve
    * - 6
      - Sensores (sin motores)
      - Ver en vivo yaw, ToF, ultrasónicos, TCRT y color. **El primer paso siempre.** Los motores están apagados.
    * - 1
      - Color
      - Ver los valores ``R``, ``G``, ``B`` y el color que detecta. Sirve para calibrar la tabla (ver :doc:`../Componentes/TCS34725`).
    * - 2
      - Lazo abierto
      - Mover los motores **a mano** con comandos, sin PID. Sirve para revisar inversión de motores, encoders y PPR (ver :doc:`../Componentes/Constantes`).
    * - 3
      - Una casilla con PID
      - Avanza una casilla y reporta qué tanto se desvió. Sirve para **tunear el PID** (ver :doc:`../PID/IntroduccionPID`).
    * - 4
      - Máquina de estados
      - El robot decide solo en cada casilla (ver :doc:`../MaquinaEstados/Introduccion`).
    * - 5
      - Maze
      - El algoritmo completo del reto.
    * - 7 a 10
      - Levels
      - Cada sección del reto por separado, o las tres seguidas.

.. important::
    Los modos están ordenados **de abajo hacia arriba**, igual que las capas del código. Prueba en este orden: **6, 1, 2, 3, 4, 5**. No pases al siguiente hasta que el anterior funcione:
    si no puedes confiar en los sensores, el PID no se puede tunear, y si no puedes confiar en el PID, la máquina de estados no tiene sentido.

Los modos con comandos
=======================

Los modos 2, 3 y los de Levels se controlan **por Serial o por Bluetooth** (el robot se llama ``RIBEYE``). Así se puede probar sin cable conectado al robot. Algunos comandos:

.. list-table::
    :header-rows: 1
    :widths: 15 15 70

    * - Modo
      - Comando
      - Efecto
    * - 2
      - ``m`` / ``b``
      - Avanzar / retroceder.
    * - 2
      - ``l`` / ``r``
      - Girar a la izquierda / derecha sobre su eje.
    * - 2
      - ``+`` / ``-``
      - Subir / bajar el PWM de 10 en 10.
    * - 2, 3
      - ``s``
      - Paro.
    * - 3
      - ``g``
      - Avanzar una casilla.
    * - 3
      - ``c``
      - Activar/desactivar el modo continuo (casilla tras casilla).
    * - 3, Levels
      - ``z``
      - Fijar el rumbo actual como el rumbo "recto".

.. note::
    En el modo 2 el robot se detiene solo a los 3 segundos de cualquier comando. Es un seguro para que, si se desconecta el Bluetooth, no se vaya corriendo.

Escáner I2C
************

.. code-block:: cpp
    :caption: Buscar dispositivos (main.cpp)

    for (uint8_t a = 1; a < 127; a++) {
        bus.beginTransmission(a);
        if (bus.endTransmission() == 0) { Serial.printf(" 0x%02X", a); n++; }
    }

Intenta hablar con **todas las direcciones posibles** y reporta las que contestan. Es lo primero que hay que hacer si un sensor "no aparece": si no sale en el escaneo, el problema es de cableado o de dirección,
no de código.
