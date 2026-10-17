#######################
Sensor laser (VL53L0X)
#######################

El VL53L0X es un sensor láser de distancia (*Time of Flight*, ToF) que mide **cuánto tarda la luz en ir y regresar**. En RIB-EYE va **al frente** y nos dice si hay una pared adelante y a qué distancia
exacta está, que es lo que necesita el robot para **centrarse antes de girar** (ver :doc:`../Subsistemas/Introduccion`). Es mucho más estable que un ultrasónico (ver :doc:`HC-SR04`), pero es más
delicado de configurar.

Este componente tiene tres problemas que resolver: **su dirección I2C**, **su arranque** y **no quedarse esperando**.

Problema 1: la dirección I2C
*****************************

El VL53L0X siempre arranca con la dirección ``0x29``. El sensor de color TCS34725 **también** usa ``0x29`` y no se puede cambiar. Dos dispositivos con la misma dirección en el mismo bus no pueden convivir.

La solución que usamos fue darle al ToF **su propio bus I2C**.

.. code-block:: cpp
    :caption: Dos buses I2C (main.cpp)

    Wire.begin(21, 22);
    Wire.setClock(100000);
    Wire.setTimeOut(10);

    Wire1.begin(5, 18);
    Wire1.setClock(100000);
    Wire1.setTimeOut(10);

La ESP32 tiene dos controladores I2C. ``Wire`` lo usan el BNO055 y el TCS34725, y ``Wire1`` el VL53L0X. Esta fue la solución cuando, en pruebas, conectar los sensores láser al mismo bus nos dio muchos problemas
(ver :doc:`../../Electronica/Pruebas/Pruebas`).

.. tip::
    La alternativa clásica es usar el pin **XSHUT** para apagar los sensores y cambiarles la dirección uno por uno al arrancar. Funciona, pero es más complicado. Con un solo ToF, un bus propio es más simple y más robusto.

Problema 2: el arranque
************************

.. code-block:: cpp
    :caption: Reinicio limpio por hardware (ToF.cpp)

    pinMode(_xShutPin, OUTPUT);
    digitalWrite(_xShutPin, LOW);
    delay(10);
    digitalWrite(_xShutPin, HIGH);
    delay(15);

El pin **XSHUT** funciona como un interruptor de encendido del sensor. Bajarlo y subirlo **reinicia el sensor por completo**, y así siempre arranca desde un estado limpio, incluso si la ESP32 se reinició a medias
mientras el sensor estaba midiendo.

Como a veces el sensor no responde a la primera, el contenedor lo reintenta:

.. code-block:: cpp
    :caption: Reintentos de arranque (RobotContainer.cpp)

    for (int attempt = 0; attempt < 5; ++attempt) {
        if (_tofL.init(Wire1)) {
            break;
        }
        delay(20);
    }

Problema 3: no quedarse esperando
**********************************

Una medición del VL53L0X tarda alrededor de **33 ms**. Si el código esperara ese tiempo en cada lectura, el PID se quedaría congelado. Por eso el sensor se pone en modo **continuo** y ``update()`` solo
pregunta "¿ya tienes algo nuevo?":

.. code-block:: cpp
    :caption: Lectura sin bloquear (ToF.cpp)

    if (_lox.isRangeComplete()) {
        uint16_t dist = _lox.readRangeResult();
        _lastStatus = (int)_lox.Status;
        if (_lox.Status != VL53L0X_ERROR_NONE) {
            ++_errCount;
            _state = ToFStates::MEASURING;
        } else {
            _currentDistanceMm = (dist > 2000) ? 2000 : dist;
            _state = ToFStates::UP;
        }
        _lastMT = millis();

``getDistance()`` siempre devuelve **la última lectura buena**, sin esperar nada. Algunos detalles:

- **Se limita a 2000 mm.** Cuando no hay nada a la vista el sensor puede devolver valores muy grandes o sin sentido; 2000 significa "nada cerca".
- **Cuenta los errores** (``getErrorCount()``) para poder ver en las pruebas si el sensor está fallando.
- **Se auto-repara:** si pasan 150 ms sin una lectura nueva, se reinicia la medición continua.

.. code-block:: cpp
    :caption: Auto-reparación (ToF.cpp)

    } else if (millis() - _lastMT > _timeout_ms) {
        _lox.stopRangeContinuous();
        _lox.startRangeContinuous();
        _lastMT = millis();
        _state = ToFStates::MEASURING;
    }

Los estados del sensor
***********************

La clase usa una pequeña máquina de estados para saber en qué situación está:

.. list-table::
    :header-rows: 1
    :widths: 20 80

    * - Estado
      - Significado
    * - ``UNINIT``
      - Todavía no se ha iniciado (o está apagado).
    * - ``MEASURING``
      - Iniciado, esperando la primera lectura buena (o recuperándose de un error).
    * - ``UP``
      - Ya tiene una lectura buena. **Solo en este estado el dato es confiable** (``isUp()``).
    * - ``ERR``
      - No se pudo iniciar.

.. important::
    El contenedor usa ``isUp()`` para decidir si confiar en el ToF. Si el sensor no está listo, ``frontWall()`` responde **"sí hay pared"** (ver :doc:`../Contenedor/Introduccion`).
    Es mejor que el robot se detenga por precaución a que avance contra una pared que no puede ver.

Cómo ajustarlo
***************

- **kFWallmm** (en ``driveConfig.h``): distancia a la que se considera que hay pared enfrente. Ajústala con una pared real y las dimensiones de la casilla.
- Después de mover o detener el robot, **espera una lectura nueva** antes de decidir (alrededor de 33 ms). Por esto el acercamiento a la pared se hace por pulsos con pausas (ver :doc:`../Subsistemas/Introduccion`).
