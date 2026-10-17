############################
Programación de componentes
############################

Es importante tener listos los métodos para todos los componentes que vayamos a utilizar, esto quiere decir que es muy conveniente
tener POO aplicada a nuestros componentes para simplemente invocarlos como objetos y utilizar sus métodos en vez de tener que
configurar 1 nuevo cada vez que querramos usarlo.

Las reglas que siguen todos los componentes
********************************************

Todos los componentes de RIB-EYE se escribieron con las mismas tres reglas. Si las entiendes aquí, el resto de las páginas se leen mucho más rápido.

1. **init() y update() separados.** ``init()`` configura el hardware una sola vez y ``update()`` se llama en cada vuelta del ciclo principal.
2. **Nada bloquea.** Un ``delay()`` o esperar a que un sensor termine detiene todo el programa, incluyendo el PID. Por eso los sensores usan la idea de
   "pregunta si ya hay una lectura nueva, y si no, sigue con tu vida" (por ejemplo ``isRangeComplete()`` o ``millis()``).
3. **Una lectura inválida se nota.** Cada sensor puede decir "esta lectura no sirve" (``isValid()``, ``isUp()``), y quien lo usa decide qué hacer.

.. important::
    Los pines y las constantes **no** viven dentro de los componentes, sino en un solo lugar (``include/constants``). Así, si cambia el cableado o hay que
    ajustar algo, se cambia un solo archivo. Mira :doc:`Componentes/Constantes`.

.. toctree::
   :caption: Componentes
   :maxdepth: 1

   Constantes y pines <Componentes/Constantes>
   Motores (con encoder) JGA-125 371 <Componentes/JGA-125371>
   Sensor de color TCS34725 <Componentes/TCS34725>
   Sensor ultrasónico HC-SR04 <Componentes/HC-SR04>
   Sensor láser VL53L0X <Componentes/VL53L0X>
   Unidad de Medicion Inercial (imu) <Componentes/BNO055>
   Sensor de línea TCRT5000 <Componentes/TCRT5000>
