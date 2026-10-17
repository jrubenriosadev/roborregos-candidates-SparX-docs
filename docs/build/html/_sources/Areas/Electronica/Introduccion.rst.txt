#####################
Área de Electrónica
#####################

El área de electrónica se encarga de que **RIB-EYE** pueda sentir lo que pasa a su alrededor y moverse: elegir los **sensores y actuadores**, alimentarlos,
conectarlos con la **ESP32** y diseñar la **PCB** donde todo se integra. Es el punto de unión entre :doc:`Mecánica <../Mecanica/Introduccion>` (donde se monta todo)
y :doc:`Programación <../Programacion/Introduccion>` (que necesita lecturas confiables para que el algoritmo funcione).

Esta sección documenta **qué componentes se probaron y cuáles se eligieron (y por qué)**, cómo se alimenta y se conecta todo, y cómo se diseñó la placa.

.. tip::
    Casi todas las decisiones de aquí nacen de un **problema encontrado en pruebas** (lecturas con ruido, un sensor que no era el esperado, un cableado mal conectado...).
    Te recomendamos leer también :doc:`Pruebas/Pruebas`: ahí están los errores que nos pasaron para que tú no los repitas.


Herramientas utilizadas
************************

1. **ESP32:** Es el microcontrolador del robot. Al ser una versión de **38 pines** alcanzaron todos los pines para los componentes sin necesidad de usar extensores.

2. **EasyEDA:** Es el software con el que se hicieron el esquemático y el diseño de la PCB.

3. **Protoboard y módulos de prueba:** Antes de soldar nada, cada componente se probó por separado y luego en conjunto con la ESP32.

.. note::
    El **BOM (lista de componentes)** del proyecto se encuentra en esta
    `hoja de cálculo <https://docs.google.com/spreadsheets/d/1u5fW0rAIrO9JiF5qdIsv7SzFx1JUm9SMHFaoMUzko4c/edit?gid=0#gid=0>`_.


Componentes finales
********************

.. list-table::
    :header-rows: 1
    :widths: 26 24 50

    * - Función
      - Componente
      - Notas
    * - Microcontrolador
      - ESP32 (38 pines)
      - Reemplazó a una ESP32-WROOM que se quemó en pruebas.
    * - Orientación (IMU)
      - BNO055
      - Reemplazó al MPU6050, cuyas lecturas no eran precisas.
    * - Distancia (frontal)
      - VL53L0X
      - Sensor láser ToF. Se comunica por I2C.
    * - Distancia (laterales)
      - Ultrasónico
      - Mejor rango a corta distancia que el VL53L0X. Su señal ECHO necesita un divisor de voltaje.
    * - Color
      - TCS34725
      - Se comunica por I2C.
    * - Motores
      - JGA25-370 con encoder
      - Buen torque. Ver :doc:`Componentes/Actuadores`.
    * - Control de motores
      - TB6612FNG
      - Un módulo por cada dos motores.
    * - Regulación de voltaje
      - LM2596
      - Alimenta la ESP32 y la lógica desde las pilas.
    * - Visión
      - Cámara con conexión CSI (Raspberry Pi)
      - Reemplazó a la idea de usar una cámara VGA OV7670.

..
    TODO (Bernardo): confirma qué sensor va en cada posición (frente/laterales) y la cantidad final de VL53L0X y ultrasónicos.
    Revisa además el modelo exacto del motor: en Programación se escribió "JGA-125 371" y aquí "JGA25-370".


Cronología
***********

.. list-table::
    :header-rows: 1
    :widths: 14 86

    * - Semana
      - Qué pasó
    * - 2
      - Primeras pruebas con ultrasónico, TCS34725 y MPU. Las lecturas del ultrasónico tienen mucho ruido, así que se buscan medidores láser. La cámara OV7670 se cambia por una CSI.
    * - 3
      - Llegan los motores JGA25-370, se elige el TB6612FNG y el LM2596. Primer esquemático.
    * - 4
      - Se prueban encoders, VL53L0X y TB6612FNG. Se cambia el MPU6050 por el BNO055. Se quema una ESP32 por cablear mal los encoders.
    * - 5
      - Primeras lecturas buenas con el BNO055 y pruebas del TCS34725. Esquemático final con JST-XH.
    * - 6
      - Todo el conjunto funcionando juntos y primeras pruebas del PID en una cancha de cartón. Se termina el diseño de la PCB.
    * - 7
      - Se pasa a cuatro motores. Se suelda la placa.

.. toctree::
    :caption: Componentes
    :maxdepth: 1

    Sensores <Componentes/Sensores>
    Motores y control de motores <Componentes/Actuadores>
    Alimentación y filtrado <Componentes/Alimentacion>

.. toctree::
    :caption: Diseño de la placa
    :maxdepth: 1

    Esquemático <Diseno/Esquematico>
    PCB <Diseno/PCB>

.. toctree::
    :caption: Pruebas
    :maxdepth: 1

    Pruebas y problemas encontrados <Pruebas/Pruebas>
