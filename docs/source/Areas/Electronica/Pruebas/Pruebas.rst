###################################
Pruebas y problemas encontrados
###################################

Esta página recopila cómo se fue probando todo el sistema y los errores que nos pasaron. Esperamos que te sirva para evitarlos.

Cancha de pruebas
******************

Para probar correctamente los **ultrasónicos** y los **VL53L0X** se necesitaba una cancha. El compañero de programación compró un **cartón** y se dividió en **cuadrantes de 30x15**,
con lo que se hizo una tira para probar las **paredes** y los **colores**.

.. note::
    Se probaron primero los encoders junto con el BNO055. En esta etapa se siguió usando el módulo **L298N** mientras se hacían las primeras pruebas del **PID** (ver :doc:`../../Programacion/PID/IntroduccionPID`).

Problemas y soluciones
***********************

.. list-table::
    :header-rows: 1
    :widths: 12 38 50

    * - Semana
      - Problema
      - Solución
    * - 2
      - Ultrasónicos con mucho ruido en sus mediciones.
      - Se buscaron alternativas láser (VL53L0X).
    * - 2
      - MPU no era un 6050 típico, incompatible con las librerías.
      - Se buscó uno compatible y finalmente se cambió por el BNO055 (semana 4).
    * - 4
      - Encoders mal conectados (positivo y negativo confundidos) **quemaron la ESP32-WROOM**.
      - Se cambió a una ESP32 de 38 pines. Resultó beneficioso: no hizo falta usar ningún extensor de pines y se simplificó la placa.
    * - 4
      - MPU6050 con lecturas imprecisas para conocer la posición del robot.
      - Se cambió por el BNO055.
    * - 5
      - El BNO055 dejó de dar lecturas.
      - Se consiguió otro y no hubo más problemas.
    * - 6
      - Los encoders fallaban y necesitaban extensiones.
      - El problema era la **distribución de pines** que se usaba. Al cambiarla todo funcionó.
    * - 6
      - Problemas al conectar los VL53L0X al bus I2C.
      - Se creó un segundo canal I2C con otros dos pines (sugerencia del compañero de programación).
    * - 6
      - Ruido de los motores y de las pilas.
      - Capacitores cerámicos de 100 nF en los motores y un electrolítico para las pilas.
    * - 7
      - El diseño anterior no funcionaba para los obstáculos.
      - Se pasó a cuatro motores con un segundo TB6612FNG (mismos pines).

Lecciones aprendidas
*********************

- **Revisa la polaridad** de todo antes de energizar. Un cable invertido en un encoder dañó una ESP32.
- **Prueba cada componente por separado** antes de juntarlos. Así se detectó rápido cuáles fallaban.
- **Si un pin falla, revisa también la distribución de pines**: no siempre es el componente.
- Un sensor puede dañarse: **ten una unidad de repuesto** de los componentes críticos (por ejemplo, el BNO055).
- Considera que el **BNO055 debe estar centrado** en el robot para que sus lecturas sirvan para el control del giro.

.. important::
    En la semana 5, con las primeras lecturas del BNO055, se vio que el diseño con **tracción trasera** no permitiría que el robot **girara sobre su propio eje**. Esto fue lo que
    llevó al rediseño mecánico descrito en :doc:`../../Mecanica/Evolucion/Version3`.

..
    TODO (Bernardo): agrega fotos de la cancha de cartón y de las pruebas, y si hay más problemas o mediciones (error del encoder, rango real de los sensores), agrégalos aquí.
