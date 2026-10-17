###################
Introducción a PID
###################

Si le mandas la misma velocidad a los dos motores, el robot **no va recto**. Un motor es un poco más fuerte que el otro, una rueda patina más, el piso no es perfecto... y al final el robot se va chueco.
La solución no es encontrar valores "perfectos" (no existen), sino **corregir sobre la marcha**: medir hacia dónde va, compararlo con hacia dónde debería ir y corregir la diferencia. A eso se le llama **control con retroalimentación**,
y el PID es la forma más común de hacerlo.

En RIB-EYE el PID se usa para dos cosas: **ir recto** y **girar un ángulo exacto**. En ambos casos el sensor es el BNO055 (ver :doc:`../Componentes/BNO055`) y la salida es una **diferencia de velocidad** entre los motores.

La idea en tres frases
***********************

Todo parte del **error**, que es la diferencia entre lo que quiero y lo que tengo (por ejemplo, "quiero estar a 0° y estoy a 5°", el error es de 5°). El PID suma tres respuestas distintas a ese error:

- **P (proporcional):** reacciona al error **de ahora**. Cuanto más lejos estoy, más fuerte corrijo.
- **I (integral):** reacciona al error **acumulado**. Si llevo mucho tiempo un poquito desviado, corrijo más.
- **D (derivativa):** reacciona a **qué tan rápido me estoy moviendo**. Sirve para frenar antes de pasarme.

.. math::

    salida = K_p \cdot error + K_i \cdot \int error \, dt - K_d \cdot \frac{d(medicion)}{dt}

``Kp``, ``Ki`` y ``Kd`` son las **ganancias**: cuánto "peso" le damos a cada una. Encontrar sus valores es a lo que se le llama **tunear** el PID, y más abajo vemos cómo se hace.

.. tip::
    Una forma de imaginarlo es conducir un coche en carretera. **P** es girar el volante más fuerte cuanto más cerca estás de la orilla. **D** es notar que te acercas muy rápido a la orilla y corregir antes.
    **I** es darte cuenta de que el coche siempre se jala un poco a la derecha y compensarlo.

Construyendo el controlador paso a paso
****************************************

No hace falta entender todo de golpe. El ``PIDController`` se construye agregando una pieza a la vez, y cada pieza resuelve **un problema que se encontró en la práctica**.

1. La base: P y los límites
============================

.. code-block:: cpp
    :caption: Constructor y término proporcional (PIDController.cpp)

    PIDController::PIDController(float kp, float ki, float kd, float minOutput, float maxOutput)

    float pT = _kp * err;

El constructor recibe las tres ganancias y **los límites de la salida**. Aquí la salida es directamente un **PWM**, así que por defecto va de -255 a 255 (el rango de los motores).
Un PID sin límites puede pedir "velocidad 5000" y eso no existe.

2. El tiempo real (dt)
=======================

.. code-block:: cpp
    :caption: Calcular dt (PIDController.cpp)

    float dt = (float)(m - lastMicros) / 1000000.0f;
    lastMicros = m;

    return update(measurement, setpoint, dt);

La integral y la derivada **dependen del tiempo** (cuánto llevo así, qué tan rápido cambió). Por eso el PID mide el ``dt`` real entre llamadas en lugar de suponer que siempre son 20 ms.
Aun así, el ciclo principal corre a un periodo fijo (ver :doc:`../Main/CicloPrincipal`) para que ese ``dt`` sea siempre parecido.

Además, la **primera** llamada no devuelve una corrección: solo guarda el tiempo y la medición y regresa 0. Sin esto, el primer ``dt`` sería enorme (desde que arrancó el robot) y daría una salida absurda.

3. La integral y el "windup"
=============================

.. code-block:: cpp
    :caption: Integral con límite (PIDController.cpp)

    integral_pp += err * dt;

    float maxI = (_ki > 0.0f) ? (_maxOutput / _ki) : 0.0f;
    integral_pp = constrain(integral_pp, -maxI, maxI);
    float iT = _ki * integral_pp;

La integral **suma el error con el tiempo**. El peligro es el *windup*: si el robot está atorado (por ejemplo, contra una pared), el error no desaparece y la integral **crece y crece**. Cuando el robot al fin se libera, esa integral
gigante lo manda disparado.

La solución es **limitar la integral** a un valor tal que su aporte nunca pase de la salida máxima (``maxOutput / ki``).

.. note::
    En RIB-EYE **Ki vale 0** (la integral no se usa). Con P y D fue suficiente. Solo conviene agregar Ki si, después de tunear P y D, el robot **se queda siempre con un pequeño error constante** que P no logra quitar.

4. La derivada: sobre la medición y filtrada
=============================================

.. code-block:: cpp
    :caption: Derivada filtrada (PIDController.cpp)

    float dMeas = _angleWrapping ? CWErr(measurement, lastMeasurement) : (measurement - lastMeasurement);
    lastMeasurement = measurement;
    lastD += kDAlpha * ((dMeas / dt) - lastD);
    float dT = -_kd * lastD;

Hay dos decisiones aquí:

- **Se deriva la medición, no el error.** Cuando cambias el objetivo de golpe (por ejemplo, "ahora gira 90°"), el error salta de un instante a otro y su derivada sería **enorme** (un "golpe" o *derivative kick*).
  La medición, en cambio, cambia suavemente porque el robot es un objeto físico. El signo menos de ``dT`` viene de esto: se frena cuando la medición se mueve rápido.
- **Se filtra.** Derivar amplifica el ruido: una lectura con pequeñas variaciones da una derivada muy nerviosa. El filtro (``kDAlpha = 0.3``) **mezcla un 30% de la lectura nueva con el 70% de lo anterior**, suavizando el resultado.

5. Sumar y limitar
===================

.. code-block:: cpp
    :caption: La salida (PIDController.cpp)

    lastOutput = constrain(pT + iT + dT, _minOuput, _maxOutput);
    return lastOutput;

Se suman las tres respuestas y se limita al rango permitido.

6. Los ángulos dan la vuelta
=============================

.. code-block:: cpp
    :caption: Error por el camino corto (PIDController.cpp)

    float PIDController::CWErr(float setpoint, float measurement) const {
      float err = setpoint - measurement;

      while (err > M_PI) { err -= 2.0f * M_PI; }
      while (err < -M_PI) { err += 2.0f * M_PI; }

      return err;
    }

Los ángulos son circulares: 359° y 1° están a solo 2° de distancia, pero restados dan 358°. Si el PID usara esa resta, el robot daría casi una vuelta completa **en la dirección equivocada**.
Con ``setAngleWrapping(true)`` el error se ajusta para que siempre sea **por el camino más corto** (entre -π y π).

.. important::
    Los ángulos del PID están en **radianes**, no en grados. El robot convierte con ``kDegToRad`` antes de usarlos. Esto importa al tunear: las ganancias ``Kp`` que verás en la tabla de abajo son "PWM por **radián** de error".

7. Empezar de cero
===================

``reset()`` borra la integral, la derivada y el tiempo guardados. Se llama **al inicio de cada movimiento nuevo** (ver :doc:`../Subsistemas/Introduccion`), porque si el PID conserva lo que "aprendió" en el movimiento
anterior, los primeros instantes del siguiente salen mal.

Cómo lo usa el robot
*********************

El ``Drive`` tiene **dos PID**, uno para cada trabajo:

.. code-block:: cpp
    :caption: Los dos PID (Drive.cpp)

    _steerPID(Cfg::kYawKp, Cfg::kYawKi, Cfg::kYawKd, -255.0f, 255.0f),
    _turnPID(Cfg::kSteerKp, 0.0f, Cfg::kSteerKd, -255.0f, 255.0f),

.. list-table::
    :header-rows: 1
    :widths: 18 22 60

    * - PID
      - Constantes
      - Qué hace
    * - ``_steerPID``
      - ``kYawKp``, ``kYawKi``, ``kYawKd``
      - **Ir recto.** Compara el yaw con el rumbo deseado y devuelve cuánto hay que **restarle a un motor y sumarle al otro**.
    * - ``_turnPID``
      - ``kSteerKp``, ``kSteerKd``
      - **Girar en su lugar.** Los dos motores giran en sentidos opuestos hasta llegar al ángulo.

Son dos PID separados porque **no tienen las mismas necesidades**: ir recto necesita correcciones pequeñas y suaves, y girar necesita arrancar con fuerza y frenar con precisión.

Los valores actuales
=====================

.. list-table::
    :header-rows: 1
    :widths: 22 12 66

    * - Constante
      - Valor
      - Qué controla
    * - ``kYawKp``
      - 220
      - Recto: fuerza de la corrección. 1° de error son unos 3.8 PWM.
    * - ``kYawKi``
      - 0
      - Recto: sin integral.
    * - ``kYawKd``
      - 6
      - Recto: amortigua el zigzag.
    * - ``kMaxSteer``
      - 40
      - Recto: tope de la corrección (se alcanza con ~10° de error).
    * - ``kSteerKp``
      - 180
      - Giro: fuerza (1° de error son unos 3.1 PWM).
    * - ``kSteerKd``
      - 12
      - Giro: frena antes de pasarse.
    * - ``kSteerMin`` / ``kSteerMax``
      - 110 / 180
      - Giro: PWM mínimo y máximo permitidos.
    * - ``kSteerTol``
      - 2.5°
      - Giro: error con el que se da por terminado.
    * - ``kSteerST``
      - 5
      - Giro: ciclos seguidos dentro de la tolerancia (5 x 20 ms) antes de aceptar el giro.

.. important::
    Estos valores son **de nuestro robot**, con sus motores, su peso y su piso. **No los copies**: sirven para tener un punto de partida, no para que los uses directamente.

Cómo tunear
************

Tunear es probar, observar y ajustar. Lo importante es hacerlo **en orden y de a una constante a la vez**.

Antes de empezar
=================

Revisa que lo demás funcione, o vas a "tunear" un problema que no es del PID:

1. Los sensores dan lecturas buenas (modo de prueba 6, ver :doc:`../Main/CicloPrincipal`).
2. Los motores giran en el sentido correcto (modo 2).
3. Conoces el **PWM mínimo con el que el robot realmente se mueve** (la *zona muerta*). Por debajo de ese valor los motores zumban pero no giran.
   En nuestro caso es alrededor de **110** (es el origen de ``kMinOutput`` y ``kSteerMin``). Un PID cuya salida cae dentro de la zona muerta **no hace nada**.

Ir recto (``kYawKp`` y ``kYawKd``)
===================================

Usa el modo 3, que avanza una casilla y **reporta cuánto se desvió del rumbo**.

1. Pon ``Ki`` y ``Kd`` en 0.
2. **Sube Kp** poco a poco. Al principio el robot sigue chueco; mientras sube, corrige más. Cuando empieza a hacer **zigzag** (oscila de un lado a otro), te pasaste. Baja Kp a más o menos **la mitad o dos tercios** de ese valor.
3. **Sube Kd** hasta que el zigzag desaparezca. Si lo subes demasiado, los motores se ponen "nerviosos" (vibran o zumban) porque el D amplifica el ruido.
4. Solo si queda un error constante, prueba una ``Ki`` muy pequeña.

Girar (``kSteerKp``, ``kSteerKd``, ``kSteerMin``)
==================================================

El modo 3 no prueba giros; para ello usa el modo 4 (la máquina de estados hace giros de 90° contra una pared) o el 5.

Con las constantes actuales hay algo que conviene entender: como ``kSteerMin`` es 110 y ``kSteerKp`` es 180, la salida del PID baja de 110 cuando el error es **menor a unos 35°** (110 / 180 rad).
Desde ahí y hasta que el error entra en la tolerancia (2.5°), el robot gira con **el PWM mínimo fijo** (+110 o -110, según el signo del error), y en ese momento se detienen los motores.

Esto significa que, en la práctica, ``Kp`` y ``Kd`` solo mandan en **la primera parte del giro** (con error mayor a ~35°), y que la precisión final la deciden ``kSteerMin``, ``kSteerTol`` y ``kSteerST``. Por eso:

- Si el robot **se pasa del ángulo** al final del giro: baja ``kSteerMin`` (llega con menos inercia, siempre que el robot aún se mueva) o sube ``kSteerKd`` (la salida baja al PWM mínimo antes, así que llega más lento).
- Si el robot **se queda corto** y no llega: sube ``kSteerMin`` (puede estar bajo la zona muerta) o baja ``kSteerTol``.
- Si el giro es **muy lento**: sube ``kSteerMax`` o ``kSteerKp`` (aceleran la primera parte) y revisa que ``kSteerMin`` no sea demasiado bajo.

Síntomas y qué hacer
=====================

.. list-table::
    :header-rows: 1
    :widths: 40 60

    * - Lo que ves
      - Qué probar
    * - Hace zigzag al ir recto
      - Bajar ``kYawKp`` o subir ``kYawKd``.
    * - Va chueco siempre al mismo lado y no corrige
      - Subir ``kYawKp``. Si un motor es más fuerte, ajustar ``kTrim`` (ver :doc:`../Subsistemas/Introduccion`).
    * - Los motores vibran o zumban
      - Bajar ``kYawKd`` o ``kSteerKd`` (están amplificando ruido).
    * - Se pasa del ángulo al girar
      - Subir ``kSteerKd`` o bajar ``kSteerMin``.
    * - No termina nunca de girar
      - Subir ``kSteerTol`` o revisar que ``kSteerMin`` supere la zona muerta.
    * - Tarda mucho en dar por terminado el giro
      - Bajar ``kSteerST``.

.. tip::
    Anota **cada cambio y su resultado**. Con tantas constantes, es muy fácil olvidar cuál fue la que mejoró (o empeoró) el comportamiento.
