########################
Subsistemas (Subsystems)
########################

La máquina de estados decide **qué** hacer ("avanza una casilla", "gira 90°"), pero alguien tiene que saber **cómo** hacerlo: qué velocidad darle a cada motor, cuándo frenar, cómo corregir si se desvía.
Ese trabajo es de un **subsistema**: una clase que agrupa los actuadores de una parte del robot junto con sus controladores, y ofrece **acciones completas** en lugar de comandos sueltos.

En RIB-EYE el único subsistema de movimiento es el ``Drive`` (``subsystems/drive``). Agrupa los dos motores (izquierdo y derecho) y los dos PID, y ofrece verbos como ``moveTile`` o ``turnToAngle``.

.. note::
    Aunque el robot tiene cuatro motores, el ``Drive`` solo maneja **dos** (uno por lado). Los dos motores de cada lado se controlan con los mismos pines de la ESP32, por lo que se mueven siempre juntos
    (ver :doc:`../../Electronica/Componentes/Actuadores`).

Dos decisiones de diseño
*************************

1. El Drive no conoce los sensores
===================================

.. code-block:: cpp
    :caption: Los sensores entran como parámetros (Drive.hpp)

    bool turnToAngle(float current_yawDeg);
    bool moveTile(float currentYaw, float distLeft, float distRight, float baseSpeed = 140.0f);

El ``Drive`` **no tiene un BNO055 ni un ultrasónico adentro**: quien lo usa le **pasa** el yaw y las distancias en cada llamada. Esto es posible gracias al contenedor (ver :doc:`../Contenedor/Introduccion`),
que es el dueño de los sensores. La ventaja es que el ``Drive`` solo se ocupa de **mover**, y se puede probar con cualquier fuente de lecturas.

2. Preparar y ejecutar (prep / step)
=====================================

Casi todos los movimientos vienen en **pares**:

.. code-block:: cpp
    :caption: Patrón prep/step (Drive.hpp)

    void prepMoveTile(float targetDistanceMeters, float targetYawDeg);
    bool moveTile(float currentYaw, float distLeft, float distRight, float baseSpeed = 140.0f);

- ``prepX(...)`` se llama **una vez**, al empezar el movimiento: fija el objetivo, pone en cero los encoders y **reinicia los PID**.
- ``X(...)`` se llama **en cada ciclo** y devuelve ``true`` cuando el movimiento **terminó**.

Es el patrón de la máquina de estados (ver :doc:`../MaquinaEstados/Introduccion`) llevado al movimiento: **nunca bloquea**. La máquina de estados llama a ``moveTile`` en cada vuelta y pasa al siguiente estado cuando devuelve ``true``.
El ``prep`` es importante porque **reinicia** los PID: si no, la integral y la derivada del movimiento anterior contaminan el nuevo (ver :doc:`../PID/IntroduccionPID`).

Avanzar una casilla (moveTile)
*******************************

Este es el movimiento más complejo. Lo vamos construyendo en capas, de la más básica a la más fina.

Capa 1: saber cuánto falta
===========================

.. code-block:: cpp
    :caption: Distancia restante (Drive.cpp)

    const float r = _rightMotor.getDistanceMeters() * Cfg::kRScale;
    const float l = r;
    const float remaining = _targetDistance - (l + r) * 0.5f;

    if (remaining <= Cfg::kStopTol) {
        stop();
        return true;
    }

La distancia viene de los **encoders** (cuántas vueltas dio la rueda por su diámetro). Se compara con el objetivo (una casilla, ``kTSize = 0.30`` m) y se **detiene un poco antes** de llegar (``kStopTol = 0.03`` m):
los motores no frenan de golpe, así que si se mandara el paro justo en 30 cm, el robot **se pasaría**. Si el robot se queda corto o se pasa de la casilla, este es el valor a ajustar.

.. note::
    **Estado actual del código:** la distancia se toma solo del encoder **derecho** (``l = r``). El izquierdo está comentado. Por eso la corrección por diferencia de encoders (capa 4) hoy da siempre 0.
    El rumbo recto lo logra el BNO055.

Capa 2: frenar antes de llegar
===============================

.. code-block:: cpp
    :caption: Desacelerar al final (Drive.cpp)

    float speed = baseSpeed;
    if (remaining < Cfg::kSlow) {
        speed = fmaxf(Cfg::kMinOutput, baseSpeed * (remaining / Cfg::kSlow));
    }

Los últimos 15 cm (``kSlow``) el robot **baja la velocidad** de forma proporcional a lo que le falta, para llegar más despacio y frenar con precisión. Pero **nunca baja de** ``kMinOutput`` (110):
por debajo de ese PWM los motores **no se mueven** (zona muerta), y el robot se quedaría parado antes de llegar.

Capa 3: ir recto con el BNO055
===============================

.. code-block:: cpp
    :caption: Corrección por yaw (Drive.cpp)

    const float imuOut = _steerPID.update(yawRad, _targetYaw + wallOffset);
    const float imuSteer = Cfg::kImu ? imuOut : 0.0f;

Aquí entra el PID de rumbo (ver :doc:`../PID/IntroduccionPID`): compara el yaw con el **rumbo objetivo** y devuelve una corrección. El ``wallOffset`` es una pequeña desviación del objetivo que viene de los ultrasónicos (capa 5);
si no hay corrección por pared, vale 0.

Capa 4: que las dos ruedas vayan igual
=======================================

.. code-block:: cpp
    :caption: Corrección por encoders (Drive.cpp)

    const float encSteer = Cfg::kEnc ? (-Cfg::kEncKp * (l - r)) : 0.0f;

Si una rueda recorre más que la otra, el robot se está curveando. Esta corrección es **proporcional a la diferencia** de lo que recorrió cada rueda. Cuando se tienen los dos encoders conectados, ayuda a ir recto
**incluso si el BNO055 tarda en notar la desviación**.

Capa 5: no chocar con las paredes
==================================

.. code-block:: cpp
    :caption: Corrección por pared (Drive.cpp)

    if (okL && okR) {
        err = distRight - distLeft;
        wall = true;
    } else if (okL) {
        err = 2.0f * (Cfg::kUTarget - distLeft);
        wall = true;
    } else if (okR) {
        err = 2.0f * (distRight - Cfg::kUTarget);
        wall = true;
    }

Con las paredes a los lados, los ultrasónicos pueden decirnos **si el robot se está pegando a un lado**:

- Con **dos paredes**, el objetivo es estar centrado (la diferencia entre las dos distancias debe ser 0).
- Con **una sola**, el objetivo es mantener una distancia fija (``kUTarget``, 9 cm) a esa pared. Se multiplica por 2 para que la corrección tenga la misma escala que con dos paredes.

En lugar de mandar esto directo a los motores, el error de pared se convierte en un **pequeño desvío del rumbo objetivo** (``wallOffset``, de 5° como máximo) y el PID de yaw hace el resto. Esto es más seguro que
dos controladores peleándose por los mismos motores: **solo uno manda** (el de yaw) y la pared solo le "mueve la meta".

Antes de usar una lectura, se protege:

- Solo se usa si la lectura es **válida y cercana** (``kUMax``, 14 cm).
- Solo se usa si el robot **ya va casi derecho** (``kUMaxYawErrDeg``, 15°). Con el robot muy torcido, la pared "se ve" distinta y la lectura engañaría al controlador.
- Se **filtra** (``kUFilterA``) y se ignora un error pequeño (``kUDeadband``, **zona muerta** de 1 cm) para no andar zigzagueando por nada.

.. note::
    **Estado actual del código:** la corrección por pared está **apagada** con ``kUltra = false`` en ``driveConfig.h``. Para probarla, cámbiala a ``true`` y tunea (ver :doc:`../PID/IntroduccionPID`).

Capa 6: mezclar y mandar a los motores
=======================================

.. code-block:: cpp
    :caption: La mezcla final (Drive.cpp)

    const float totalSteer = constrain(encSteer + imuSteer, -Cfg::kMaxSteer, Cfg::kMaxSteer);

    setOpenLoop(constrain((int)(speed + totalSteer + Cfg::kTrim), 0, 255), constrain((int)(speed - totalSteer), 0, 255));

Las correcciones se **suman** y se limitan a ``kMaxSteer`` (para que ninguna corrección exagerada deje a un motor casi parado). La velocidad base se **suma a un motor y se resta al otro**:
si el robot se va a la izquierda, el motor izquierdo acelera y el derecho frena, y el robot se endereza.

- ``kTrim`` es una constante que se **suma siempre al motor izquierdo**. Compensa que un motor sea un poco más fuerte que el otro, para que "recto" no empiece ya chueco.
- ``kBSpeed`` (140) es la velocidad base. A más velocidad, el robot es más rápido pero **más difícil de controlar**.

Cómo ajustar moveTile
======================

.. list-table::
    :header-rows: 1
    :widths: 35 65

    * - Lo que ves
      - Qué ajustar
    * - Se pasa de la casilla
      - Subir ``kStopTol`` (frena antes) o bajar ``kBSpeed``.
    * - Se queda corto
      - Bajar ``kStopTol`` o revisar el diámetro de la rueda (ver :doc:`../Componentes/Constantes`).
    * - Se frena y no llega
      - ``kMinOutput`` está por debajo de la zona muerta: súbelo.
    * - Siempre se va hacia un lado
      - Ajustar ``kTrim`` y revisar ``kYawKp`` (ver :doc:`../PID/IntroduccionPID`).
    * - Hace zigzag
      - Bajar ``kYawKp`` o subir ``kYawKd``.

Girar en su lugar (turnToAngle)
********************************

.. code-block:: cpp
    :caption: Giro con PID (Drive.cpp)

    float cmd = constrain(out, -Cfg::kSteerMax, Cfg::kSteerMax);
    if (fabsf(cmd) < Cfg::kSteerMin) {
        cmd = copysignf(Cfg::kSteerMin, err);
    }

    setOpenLoop((int)cmd, (int)-cmd);

El giro usa el otro PID (``_turnPID``). Lo interesante está en la última línea: ``cmd`` y ``-cmd``. Un motor avanza y el otro retrocede **con la misma fuerza**, así que el robot **gira sobre su propio centro**.
Esto es justo lo que pidió programación y lo que motivó el rediseño del chasis (ver :doc:`../../Mecanica/Evolucion/Version3`).

Como ya se vio, la salida se **limita por abajo** (``kSteerMin``) para que nunca caiga en la zona muerta del motor.

.. code-block:: cpp
    :caption: Cuándo se da por terminado (Drive.cpp)

    if (errDeg < Cfg::kSteerTol) {
        stop();
        if (++_settleCount >= Cfg::kSteerST) {
            _settleCount = 0;
            return true;
        }
        return false;
    }

    _settleCount = 0;

No basta con **tocar** el ángulo objetivo: el robot puede llegar, pasarse un poco y volver. Por eso se exige estar **dentro de la tolerancia** (``kSteerTol``, 2.5°) durante varios ciclos seguidos (``kSteerST``, 5 ciclos = 100 ms)
antes de declarar que el giro terminó. Si en algún momento sale de la tolerancia, el contador se reinicia.

Acercarse a la pared (approachFront)
*************************************

Antes de un giro real, si hay una pared enfrente dentro de la casilla, el robot **se acerca a ella** hasta quedar a ``kTurnDist`` (55 mm) para que el giro empiece **siempre desde el mismo punto**, centrado en la casilla.

Lo curioso es **cómo** se acerca: por **pulsos** y no de forma continua.

.. code-block:: cpp
    :caption: Acercamiento pulsado (Drive.cpp)

    const float err = mm - Cfg::kTurnDist;
    if (fabsf(err) <= Cfg::kApproachTolMm) {
        stop();
        return true;
    }

    const int p = err > 0 ? (int)Cfg::kApproachPwm : -(int)Cfg::kApproachPwm;
    _apprPulse = constrain((unsigned long)(fabsf(err) * Cfg::kApproachMsPerMm), Cfg::kApproachPulseMinMs, Cfg::kApproachPulseMaxMs);
    setOpenLoop(p, p);
    _apprDriving = true;

Se avanza (o retrocede) **un pulso corto**, se **detiene**, se **espera** una lectura fresca del ToF y se repite. Hay dos razones:

1. **Un PWM bajo no mueve los motores** (zona muerta). Las primeras pruebas con PWM de 20 no hacían nada, así que se usa un PWM suficiente (``kApproachPwm``, 110) pero **por poco tiempo**.
2. **El ToF tarda ~33 ms** en dar una lectura nueva (ver :doc:`../Componentes/VL53L0X`). Si el robot avanza sin parar, mide una distancia vieja y se pasa. La pausa (``kApproachPauseMs``, 120 ms) asegura lecturas frescas.

La duración de cada pulso es **proporcional al error** (``kApproachMsPerMm``, 3 ms por milímetro), limitada entre 30 y 150 ms. Cuanto más lejos está, más largo el pulso.

.. tip::
    Si el robot **se pasa** al acercarse, baja ``kApproachMsPerMm``. Si **se queda corto** y tarda mucho, súbelo. Si nunca se acerca, revisa que ``kApproachPwm`` supere la zona muerta.

El acercamiento tiene **tres salidas** para no quedarse atorado: llegar a la distancia (con ``kApproachTolMm`` de tolerancia, 8 mm), que la pared esté demasiado lejos para ser de esta casilla (``kApproachMaxMm``, 120 mm),
o un tiempo límite (``kApproachMillis``, 2.5 s).

Reversa recta (moveBack)
*************************

``moveBack`` es la versión en reversa de ``moveTile``: mantiene el rumbo con el PID de yaw mientras retrocede cierta distancia. Se usa en el reto Levels (retroceder al ver una línea blanca y entrar de reversa por la pelota).
Es más sencilla porque **no desacelera** y no usa corrección por pared.

Lazo abierto (setOpenLoop)
***************************

``setOpenLoop(izq, der)`` manda un PWM directo a cada motor, **sin ningún controlador**. Se usa en el modo de prueba 2 y dentro de los demás movimientos como la última línea que realmente mueve los motores.
Es la única función de ``Drive`` que escribe a los motores; todo lo demás calcula **qué** PWM mandar.
