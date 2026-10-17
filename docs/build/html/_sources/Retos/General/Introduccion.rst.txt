#####################
Reglas generales
#####################

Antes de ver cada reto, hay reglas que aplican a **los dos** y que condicionan casi todas las decisiones de diseño del robot. Esta página es un resumen del reglamento de **Candidates 2026, categoría Principiantes**
(versión del 28 de agosto de 2026). Si hay alguna duda, **el reglamento oficial y el juez tienen la última palabra**.

.. warning::
    Las reglas están sujetas a cambios. Esta página resume la versión del 28 de agosto de 2026; revisa siempre el reglamento más reciente.

Cómo es una ronda
******************

La competencia tiene **tres rondas**. En cada una, el equipo pasa por **la pista A (Maze)** y **la pista B (Levels)**.

.. list-table::
    :header-rows: 1
    :widths: 30 70

    * - Concepto
      - Regla
    * - Tiempo por pista
      - **8 minutos:** los primeros **2 son de calibración** y los **6 restantes** son de ronda.
    * - Calibración
      - Se puede modificar electrónica, mecánica o programación, y probar el robot en la pista. Los puntos que se obtengan **no cuentan**.
    * - Más calibración
      - Se puede pedir más tiempo, pero **se descuenta del tiempo de ronda**. El tiempo de calibración que sobra no se transfiere.
    * - Durante la ronda
      - **Prohibido** cualquier modificación al robot.
    * - Empate
      - Gana el equipo con **menor tiempo ocupado** en las rondas.

.. tip::
    Con solo 6 minutos de ronda, **la velocidad importa**, pero un robot que se atora o se equivoca pierde más tiempo que uno lento y seguro.

Términos que vas a ver
***********************

.. list-table::
    :header-rows: 1
    :widths: 22 78

    * - Término
      - Significado
    * - **Unidad**
      - Cuadrícula individual de **30 x 30 ± 2 cm**. Es la medida estándar de las pistas.
    * - **Checkpoint**
      - Punto al que el robot regresa si hay *lack of progress*. Para que sea válido, **al menos el 50% del robot** debe estar dentro de la casilla.
    * - **Obstáculos**
      - Paredes, *speedbumps*, escaleras y rampas.
    * - **Lack of progress**
      - El robot no puede avanzar o está atorado, y hay que **reiniciar su posición**.
    * - **Calibración**
      - Ajuste de código, sensores, actuadores y posición del robot antes de empezar la ronda.

Las pistas
***********

- Hay **cuatro pistas**: dos pistas A y dos pistas B. **No están conectadas** entre sí.
- Las paredes miden **mínimo 15 cm de alto**, y está **prohibido pasar por encima de ellas**.
- Cada pista tiene **al menos un checkpoint** (el final cuenta), y los checkpoints **dan puntos**.
- Si el robot no ha llegado a ningún checkpoint, **la casilla de inicio cuenta como uno**.
- Entre unidades de distinto color hay una **separación blanca**.
- Las dimensiones de las pistas son las mismas en todas las rondas. **Cambia la distribución**: checkpoints, obstáculos, colores y objetos a encontrar.
- Una vez que el robot completa una pista, **no puede regresar** al inicio ni a ningún checkpoint, y termina la ronda.
- Los tonos de los colores **pueden variar** entre pistas.

.. important::
    Como todo puede cambiar de posición entre rondas, **el robot no puede depender de un mapa fijo**. Por eso el robot debe descubrir la pista en el momento (ver :doc:`../Maze/Introduccion`).

Lack of progress
*****************

Ocurre cuando:

- **El capitán lo declara.**
- El robot o algún miembro del equipo **daña la pista**.
- Alguien del equipo (incluido el capitán) **toca la pista o el robot sin permiso del juez**.

Cuando pasa, el capitán **apaga o reinicia el robot siempre del mismo modo** (se le dice al juez al inicio de la ronda) y lo coloca en **el checkpoint más reciente** (o en el inicio si no hay ninguno).
**El tiempo sigue corriendo.** No se puede cambiar nada del robot.

Las reglas especiales de la pista B (tres *lack of progress* en una misma sección, y qué pasa con la pelota) se explican en :doc:`../Levels/Introduccion`.

El robot
*********

- Debe caber en **una Unidad (30 x 30 cm) y 30 cm de alto**: las dimensiones se limitan al tamaño de una Unidad **en los tres ejes**.
- **Sin restricciones** de sensores, actuadores, baterías, llantas, microcontroladores o material. El peso es libre.
- Debe ser **totalmente autónomo**: está prohibido interactuar con él después de empezada la ronda (la única excepción es el capitán en un *lack of progress*).
- Está prohibida la **comunicación** de los participantes con el robot (por ejemplo por Bluetooth). La comunicación **entre componentes del propio robot** sí está permitida.
- Debe estar **hecho por el equipo**. No se permiten robots ya hechos, kits de desarrollo ni kits educativos (como Lego Mindstorms o Spike).
- Está prohibido usar herramientas que **dañen o modifiquen** la pista.
- Un robot con **80% de similitud** (diseño y programación) a otro **descalifica a ambos**.

Prohibiciones importantes
**************************

.. list-table::
    :header-rows: 1
    :widths: 30 70

    * - Prohibido
      - Detalle
    * - **Pre-mapeo**
      - No se puede guardar la configuración de la pista para la ronda en curso, **incluyendo la localización de paredes y colores**.
    * - **Código copiado**
      - Está prohibido usar código ya hecho por otra persona o bajado de internet para la **lógica y movimientos** de las pistas. Todos los integrantes deben conocer el algoritmo.
    * - **Plagio**
      - Los entregables deben ser de autoría propia. Se pueden consultar recursos en línea **si se citan y justifican**.
    * - **Sonido**
      - La emisión de sonido está prohibida (por ejemplo, para indicar colores).

.. note::
    Sobre el uso de inteligencia artificial: RoBorregos **no promueve** usarla para desarrollar la programación, los diseños mecánicos o los esquemáticos de electrónica. El equipo debe poder **explicar,
    modificar y justificar** sus decisiones sobre el robot, el código, el diseño y las bitácoras.

Quién puede tocar qué durante la ronda
***************************************

- Solo **dos personas** pueden estar en la cancha al mismo tiempo durante la ronda.
- El robot solo lo puede manipular (encender, reiniciar, reposicionar) **el capitán**.
- El capitán puede **declarar la ronda terminada** en cualquier momento, y se conservan los puntos obtenidos.

Seguimiento y entregables
**************************

Los equipos asisten semanalmente a **sesiones de seguimiento** con sus mentores y llevan una **bitácora** (de formato libre) con evidencias de su trabajo.

.. list-table::
    :header-rows: 1
    :widths: 20 80

    * - Área
      - Entregables
    * - Programación
      - Código en un repositorio de GitHub y bitácora.
    * - Electrónica
      - Esquemático (EasyEDA, Altium, Proteus o KiCad), **BOM** con links a los datasheets y bitácora. Opcionales: diseño y manufactura de PCB, placa perforada.
    * - Mecánica
      - Diseño CAD del robot (partes y ensamblajes), imágenes del robot y del CAD desde distintos puntos de vista, y bitácora. Opcionales: dibujos técnicos y BOM.

Esta documentación existe, en parte, para eso: ver :doc:`../../Areas/Programacion/Introduccion`, :doc:`../../Areas/Electronica/Introduccion` y :doc:`../../Areas/Mecanica/Introduccion`.

Los retos
**********

- :doc:`../Maze/Introduccion` (pista A).
- :doc:`../Levels/Introduccion` (pista B).
