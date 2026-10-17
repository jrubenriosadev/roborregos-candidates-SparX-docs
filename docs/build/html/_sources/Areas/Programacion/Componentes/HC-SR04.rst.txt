#############################
Sensor Ultrasónico (HC-SR04)
#############################

Los ultrasónicos van a los **lados** del robot y nos dicen si hay una pared a la izquierda o a la derecha (ver :doc:`../../Electronica/Componentes/Sensores` para saber por qué se
eligieron para los lados). Son baratos y fáciles de usar, pero tienen un problema que condiciona todo el código: **son ruidosos y bloquean** mientras esperan el eco.

Este componente se construye en dos partes: la clase ``Ultrasonic`` (que habla con el sensor) y el ``WallFilter`` (que limpia el ruido).

Cómo funciona el sensor
************************

1. Se manda un pulso de 10 µs al pin **TRIGGER**.
2. El sensor emite ultrasonido y pone el pin **ECHO** en alto **mientras espera el rebote**.
3. El tiempo que ECHO estuvo en alto es el viaje de ida y vuelta del sonido.

.. code-block:: cpp
    :caption: Convertir el eco en distancia (Ultrasonic.cpp)

    unsigned long duration = pulseIn(echoPin, HIGH, maxTimeoutUs);

    if (duration == 0 || duration < 116) {
        valid = false;
        lastDistance = maxDistanceCm;
    } else {
        valid = true;
        lastDistance = (duration * 0.0343f) / 2.0f;
    }

El sonido viaja a 0.0343 cm por microsegundo y la distancia se divide entre 2 porque el tiempo medido es de **ida y vuelta**.

Tres decisiones que importan
*****************************

1. Límite de distancia
=======================

.. code-block:: cpp
    :caption: Timeout según el rango máximo (Ultrasonic.cpp)

    maxTimeoutUs = (unsigned long)(maxDistanceCm * 2.0f / 0.0343f) + 3000UL;

``pulseIn`` **detiene el programa** hasta que llega el eco o se acaba el timeout. Si el sensor no encuentra nada (no hay pared) y no le ponemos un límite, el programa se queda esperando
(por defecto, hasta 1 segundo), y durante ese tiempo el PID no corre y el robot va "a ciegas".

Por eso se calcula un timeout **a partir de la distancia máxima** que nos interesa. Con ``maxRange = 15`` cm (``pinsConfig.h``), el bloqueo máximo es de unos **3.9 ms**.
Más allá de 15 cm no hay pared que nos importe, así que no tiene sentido esperar.

2. Una lectura inválida no es "cero"
=====================================

Si no hay eco, o llega demasiado rápido (menos de 116 µs, unos 2 cm, que es ruido o algo pegado al sensor), la lectura se marca como **inválida** y la distancia se deja en el valor máximo.

Esto es muy útil: **sin eco = no hay nada cerca = no hay pared**. Con el rango en 15 cm y el umbral de pared también en 15 cm (``kSWallcm``), una lectura sin eco **nunca cuenta como pared**.

3. Nunca leer los dos al mismo tiempo
======================================

Si los dos ultrasónicos disparan a la vez, cada uno puede escuchar el eco del otro (**diafonía**) y la lectura es basura. En el contenedor se **alternan** (izquierdo, derecho, izquierdo...)
cada 40 ms, por lo que cada sensor se lee cada 80 ms (ver :doc:`../Contenedor/Introduccion`).

El filtro de pared (WallFilter)
********************************

Aun con todo lo anterior, de vez en cuando una lectura sale **muy mal** (un pico), y un pico puede hacer que el robot crea que desapareció una pared. Para eso está ``WallFilter``.

.. code-block:: cpp
    :caption: Mediana de las últimas 3 lecturas (WallFilter.hpp)

    float a = _h[0], b = _h[1], c = _h[2];
    if((a >= b && a <= c) || (a <= b && a >= c)) return a;
    if((b >= a && b <= c) || (b <= a && b >= c)) return b;
    return c;

La idea es guardar las **últimas 3 lecturas válidas** y devolver **la del medio** (la mediana). Un pico aislado nunca es la del medio, así que desaparece.

Por qué mediana y no promedio
==============================

Un promedio **arrastra** el pico: (10, 10, 100) promedia 40. La mediana ignora el valor extremo: (10, 10, 100) da 10. Además es muy barato de calcular (sin arreglos grandes ni ordenamientos).

Qué pasa cuando no hay lecturas
================================

.. code-block:: cpp
    :caption: Olvidar el historial (WallFilter.hpp)

    if(_miss < 255) _miss++;
    if(_miss >= kMax) { _n = 0; _i = 0; }

Si llegan **2 lecturas inválidas seguidas** (``kMax = 2``), el filtro **olvida todo su historial**. Así, cuando la pared desaparece de verdad, el robot no sigue creyendo en una distancia vieja.
Mientras tenga menos de 3 datos, devuelve el único que tiene (o el promedio de 2).

.. tip::
    Si ves que el robot "tarda" en darse cuenta de que se abrió una pared, el filtro está reteniendo demasiado. Si ves que se confunde con picos, está reteniendo muy poco.
    ``kMax`` y el tamaño del historial (3) son los números a ajustar.

Cómo ajustarlo
***************

- **maxRange:** si el sensor "ve" paredes que no son (por ejemplo la pared del carril de al lado), baja el rango. Si no detecta la pared propia, súbelo.
- **kSWallcm:** distancia a la que se considera pared. Mídela con una pared real a la distancia a la que el robot va normalmente.
- Usa el modo de **prueba de sensores** (ver :doc:`../Main/CicloPrincipal`) para ver las lecturas en vivo mientras mueves una pared con la mano.

.. important::
    El pin ECHO entrega 5 V, pero la ESP32 solo soporta 3.3 V. Recuerda el divisor de voltaje de la placa (ver :doc:`../../Electronica/Componentes/Sensores`).
