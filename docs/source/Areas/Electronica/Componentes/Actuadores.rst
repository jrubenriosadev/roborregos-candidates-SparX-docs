##############################
Motores y control de motores
##############################

Motores
********

En la semana 3 se consiguieron los motores **JGA25-370**, que cuentan con **encoder** y un excelente **torque**. Los encoders son importantes porque
programación necesitaba una forma de comprobar que las mediciones tuvieran el **error mínimo** (ver :doc:`../../Programacion/Componentes/JGA-125371`).

.. note::
    En semana 4 el avance se frenó porque **las llantas de los motores no llegaron**, lo que retrasó las pruebas de los encoders. Aun así se pudo comprobar qué tan precisos son para dar *feedback*.

Puente H
*********

Al inicio se probó con un **puente H regular** (módulo L298N), pero la idea fue usar el módulo **TB6612FNG**, diseñado para **dos motores**.

Hay que tener en cuenta que el TB6612FNG:

- **No tiene salida para alimentar la ESP32.**
- Necesita una alimentación de **5 V** para operar.

Por eso se usa un regulador de voltaje (ver :doc:`Alimentacion`).

.. list-table::
    :header-rows: 1
    :widths: 20 40 40

    * -
      - L298N
      - TB6612FNG
    * - Motores por módulo
      - 2
      - 2
    * - Uso en el proyecto
      - Pruebas iniciales y como referencia para diseñar el esquemático y la placa
      - Módulo final de control de motores

Cuatro motores
***************

En la semana 7 se tomó la decisión de **usar cuatro motores**, porque el diseño anterior no funcionaba para los **obstáculos de la cancha** (ver :doc:`../../Mecanica/Evolucion/Version3`).

Para eso se agregó **un segundo módulo TB6612FNG** que usa **los mismos pines** que el primero, por lo que **no se necesitaron más pines** de la ESP32:

- Los motores **derechos** se controlan juntos, y los **izquierdos** también.
- **No se conectaron los encoders** de los dos motores extra, así que tampoco se usaron más pines.

.. important::
    Primero se consideró usar un **shift register** (con tres pines da ocho salidas digitales, suficientes para los motores). Se descartó porque
    es **más sencillo sincronizar los motores de cada lado** y no hace falta escribir código innecesario.

Conexiones
***********

- Los **motores** y las **pilas** se conectan con **borneras**, que evitan que se suelten.
- Cada motor lleva un **capacitor cerámico de 100 nF** entre sus terminales para reducir el ruido hacia el módulo TB6612FNG (no tiene polaridad).
- Los módulos puente H se conectan a la placa con **dupont**.

.. warning::
    En semana 4 se confundió el positivo con el negativo del encoder de un motor, y **se quemó la ESP32-WROOM** que se usaba originalmente. Revisa dos veces la polaridad antes de energizar.

..
    TODO (Bernardo): agrega la tabla de pines de motores/encoders y las especificaciones del motor (voltaje, rpm, torque).
