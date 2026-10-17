############################
Contenedor (RobotContainer)
############################

Imagina que cada parte del código que necesita un sensor lo crea por su cuenta: la máquina de estados crea su BNO055, el ``Drive`` crea otro, el maze crea otro más. Muy pronto tendrías
**tres objetos hablando con el mismo sensor**, peleándose por el bus I2C y dando lecturas distintas. Y si cambias un pin, tendrías que buscarlo en todos lados.

El ``RobotContainer`` resuelve esto con una idea simple: **un solo dueño de todo el hardware**. Es una clase que:

1. **Crea** todos los sensores y el ``Drive`` **una sola vez**.
2. Los **inicializa** en el orden correcto (``init()``).
3. Los **actualiza** en cada ciclo (``update()``).
4. Le da al resto del código **lecturas con sentido**, no datos crudos ("hay pared al frente", no "el ToF dice 87").

.. important::
    El resto del programa (máquina de estados, maze, levels) **recibe una referencia al contenedor** y le pide cosas. Nunca crea sensores propios. Esto se llama *inyección de dependencias*,
    y la ventaja es que si mañana cambias un sensor, solo tocas el contenedor.

Construyéndolo
***************

1. Dueño de todo
=================

.. code-block:: cpp
    :caption: Quién vive en el contenedor (RobotContainer.hpp)

    private:
        Drive _drive;
        Bno _bno;
        ColorSensor _tcs;
        ColorDetected _lastColor;
        Ultrasonic _leftUltra, _rightUltra;
        ToF _tofL;
        LineSensor _lineL, _lineR;

El contenedor **tiene** a todos los componentes como atributos privados. Quien lo use no puede tocarlos directamente: solo pide lo que necesita a través de métodos como ``getBno()`` o ``frontWall()``.

2. El cableado vive en el constructor
======================================

.. code-block:: cpp
    :caption: Conectar cada componente a sus pines (RobotContainer.cpp)

    _leftUltra(Pins::Ultrasonic::kLeftEcho, Pins::Ultrasonic::kLeftTrigger, Pins::Ultrasonic::maxRange),
    _rightUltra(Pins::Ultrasonic::kRightEcho, Pins::Ultrasonic::kRightTrigger, Pins::Ultrasonic::maxRange),
    _tofL(Pins::ToF::kLeftXShut, Pins::ToF::kLeftAddr),

Los pines vienen de ``pinsConfig.h`` (ver :doc:`../Componentes/Constantes`). El contenedor es **el único lugar donde se juntan** un componente y su pin.

3. Inicializar en orden
========================

.. code-block:: cpp
    :caption: Arranque (RobotContainer.cpp)

    _tofL.powerOff();
    delay(10);

    for (int attempt = 0; attempt < 5; ++attempt) {
        if (_tofL.init(Wire1)) {
            break;
        }
        delay(20);
    }

    // Sin esto el primer update() intentaba "ponerse al dia" con millis() leyendo los ultrasonicos seguidos (diafonia)
    _lastUltraToggle = millis();

El ``init()`` hace cosas que **dependen del orden**:

- El ToF se apaga primero y luego se inicia con **reintentos**, porque a veces no responde a la primera (ver :doc:`../Componentes/VL53L0X`).
- Si un sensor falla al iniciar, **el programa no se cae**: cada componente recuerda si está listo (``isUp()``) y el resto del código decide qué hacer.
- ``_lastUltraToggle`` se reinicia al final: ver más abajo por qué importa.

4. Actualizar todo en cada ciclo
=================================

.. code-block:: cpp
    :caption: Actualización (RobotContainer.cpp)

    void RobotContainer::update() {
        _bno.update();
        _tofL.update();
        _lineL.update();
        _lineR.update();

``update()`` se llama **una vez por ciclo** desde ``main`` (ver :doc:`../Main/CicloPrincipal`), antes de cualquier lógica. Así, cuando la máquina de estados pregunta "¿hay pared?", la respuesta está fresca.

.. note::
    Fíjate que ``_tcs`` y los ultrasónicos **no** aparecen aquí. El de color se lee solo cuando alguien lo pide (``getColorDetected()``) y los ultrasónicos tienen su propio ritmo, que vemos ahora.

5. Ultrasónicos por turnos
===========================

.. code-block:: cpp
    :caption: Un ultrasónico a la vez (RobotContainer.cpp)

    unsigned long c = millis();
    if (c - _lastUltraToggle >= 40) {
        _lastUltraToggle += 40;

        if (_readToggle) {
            float d = _leftUltra.readDistance();
            _leftFilter.push(d, _leftUltra.isValid());
        } else {
            float d = _rightUltra.readDistance();
            _rightFilter.push(d, _rightUltra.isValid());
        }
        _readToggle = !_readToggle;
    }

Esto resuelve **tres problemas** con unas cuantas líneas:

1. **Diafonía:** nunca se leen los dos a la vez (``_readToggle`` va alternando), así que uno no escucha el eco del otro.
2. **Ritmo constante:** se suma ``+= 40`` en lugar de asignar ``= millis()``, para que el ritmo promedio sea siempre de una lectura cada 40 ms. La contrapartida es que, si el programa se atrasa mucho, intentaría ponerse al corriente leyendo seguidos (otra vez diafonía). Por eso en ``init()`` se reinicia ``_lastUltraToggle``.
3. **Limpieza:** cada lectura pasa por su ``WallFilter`` (ver :doc:`../Componentes/HC-SR04`) antes de que nadie la use.

Cada ultrasónico se lee entonces cada 80 ms.

Preguntas, no números
**********************

La parte más valiosa del contenedor son los métodos que **convierten lecturas en decisiones**:

.. code-block:: cpp
    :caption: Paredes (RobotContainer.cpp)

    bool RobotContainer::frontWall() const {
        if (!tofReady()) return true;
        return _tofL.getDistance() < DriveConfig::kFWallmm;
    }

    bool RobotContainer::leftWall() const {
        return getLeftDistance() < DriveConfig::kSWallcm;
    }

Cuando la máquina de estados pregunta ``frontWall()`` no le importa si la respuesta viene de un ToF, de un ultrasónico o de una cámara. Si mañana cambiamos el sensor, **ese código no cambia**.

Y fíjate en la **primera línea** de ``frontWall()``: si el ToF no está listo, responde "sí hay pared". Es un **valor por defecto seguro**: ante la duda, el robot prefiere detenerse que avanzar a ciegas.

El mismo criterio se aplica a los ultrasónicos:

.. code-block:: cpp
    :caption: Sin lectura válida (RobotContainer.cpp)

    float RobotContainer::getLeftDistance() const { return _leftFilter.valid() ? _leftFilter.value() : Pins::Ultrasonic::maxRange; }

Si el filtro no tiene datos buenos, regresa el rango máximo, que equivale a "no veo nada cerca" (ver :doc:`../Componentes/HC-SR04`).

El color
*********

El contenedor también convierte el RGB del sensor en un **nombre de color** (``ColorDetected``) comparándolo con una tabla de referencia. Cómo funciona y cómo calibrarla está en :doc:`../Componentes/TCS34725`.

.. code-block:: cpp
    :caption: Pedir el color (RobotContainer.cpp)

    ColorDetected RobotContainer::getColorDetected() {
        RGB c;
        if (_tcs.readAsync(c, 100)) {
            _lastRGB = c;

Observa que ``getColorDetected()`` **lee el sensor solo si ya toca** (cada 100 ms) y, mientras tanto, regresa **el último color conocido**. Quien lo llama puede hacerlo en cada ciclo sin preocuparse.

Cuándo agregar algo al contenedor
**********************************

Una buena regla para decidir si algo va en el contenedor:

- **Sí:** es un sensor o actuador, o una pregunta sobre el entorno ("¿hay pared a la izquierda?").
- **No:** es una decisión del robot ("gira a la derecha"). Eso es de la máquina de estados (ver :doc:`../MaquinaEstados/Introduccion`).

Si el contenedor empieza a **decidir** cosas, se vuelve imposible de reusar entre retos.
