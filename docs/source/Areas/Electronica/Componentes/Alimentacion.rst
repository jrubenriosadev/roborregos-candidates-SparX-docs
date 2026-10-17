#########################
Alimentación y filtrado
#########################

El robot se alimenta con **pilas**, y de ahí salen distintos voltajes para la ESP32, la lógica de los módulos y los motores. Esta página explica cómo se regula
y se filtra el ruido eléctrico.

Regulador de voltaje (LM2596)
******************************

Como el **TB6612FNG** no puede alimentar a la ESP32 y además necesita **5 V**, se utiliza un regulador **LM2596**:

- Acepta un voltaje de entrada de hasta **40 V**.
- Su voltaje de salida es **variable**, gracias a un potenciómetro.
- Soporta hasta **3 A**, que son más que suficientes para alimentar a todos los componentes, como la ESP32.

.. tip::
    Como la salida se ajusta con un potenciómetro, **mide la salida con un multímetro antes de conectar** la ESP32 o cualquier sensor.

Filtrado de ruido
******************

Los motores y las pilas generan ruido eléctrico que puede afectar a los módulos y las lecturas. Para reducirlo se agregaron:

1. **Capacitores cerámicos de 100 nF** en las conexiones de cada motor. No tienen polaridad, y reducen el ruido de los motores hacia los módulos TB6612FNG.
2. **Un capacitor electrolítico** para bajar el ruido de las pilas, y así reducir al máximo el que puedan dar los componentes que manejan más cambios de voltaje.

.. important::
    A diferencia de los cerámicos, los capacitores electrolíticos **sí tienen polaridad**. Si lo conectas al revés se puede dañar.

Conectores
***********

- **JST-XH** para todos los sensores (ver :doc:`../Diseno/PCB`).
- **Borneras** para los motores y las pilas.

..
    TODO (Bernardo): agrega el valor/voltaje del capacitor electrolítico, el tipo y voltaje de las pilas, y el voltaje al que se ajustó el LM2596.
