#####################
Área de Mecánica
#####################

El área de mecánica es la encargada de diseñar y construir el cuerpo de **RIB-EYE**: el chasis, la forma en que se mueve y los mecanismos
que le permiten resolver los retos. Todo lo que diseñamos tiene que convivir con el trabajo de **programación** y **electrónica**, así que
casi cada decisión de esta sección nace de una necesidad de alguna de las otras dos áreas (por ejemplo, poder girar sobre su propio eje para que
funcione el algoritmo del maze).

Esta sección documenta **cómo evolucionó el diseño**, **qué decisiones se tomaron y por qué**, y los mecanismos que terminaron (o no) en el robot final.

.. tip::
    Si estás empezando, te recomendamos leer primero las páginas de **Evolución del diseño** en orden, ya que cada versión
    nace de los problemas que se encontraron en la anterior.


Herramientas utilizadas
************************

1. **OnShape:** Es un software de diseño CAD que corre desde el navegador. Con él se hicieron todas las piezas, los ensambles y los sketches
de distancias. Al ser en línea, el diseño se puede consultar y compartir fácilmente con el resto del equipo.

2. **RDWorks:** Es el software con el que se preparan los archivos para la máquina de corte láser. Lo utilizamos para cortar las piezas de los
prototipos en **MDF**.

3. **MDF:** Material con el que se cortaron los prototipos. Es barato y rápido de cortar, lo que nos permitió probar varias versiones
sin gastar demasiado tiempo ni material.


Flujo de trabajo
*****************

El proceso que seguimos en cada versión fue más o menos el mismo:

1. Se define con el equipo qué necesita el robot (componentes, medidas, retos a resolver).
2. Se diseña la pieza o el ensamble completo en **OnShape**.
3. Se corta en **MDF** con el láser (RDWorks) y se ensambla.
4. Programación y electrónica lo prueban y regresan retroalimentación.
5. Se rediseña con base en lo que se encontró.

.. important::
    El diseño en CAD **debe incluir representaciones de los componentes electrónicos** (sensores, controlador, baterías, etc.).
    Si no se agregan, es muy fácil que haya choques entre piezas mecánicas y eléctricas, o desconexiones inesperadas cuando ya está todo ensamblado.


Resumen de versiones
*********************

.. list-table::
    :header-rows: 1
    :widths: 18 14 68

    * - Versión
      - Fecha
      - Qué cambió
    * - Prototipo inicial
      - 3/9/26 - 9/9/26
      - Primera base para que programación y electrónica pudieran empezar a probar. Primeros intentos de un sistema de orugas.
    * - Versión 1
      - 17/9/26
      - Diseño con más detalle y perforaciones para fijar componentes. Se descartan temporalmente las orugas.
    * - Versión 2
      - 18/9/26 - 27/9/26
      - Rediseño mayor: motores finales, placa superior, 4 ruedas con solo 2 motores (en lugar de rueda loca) y primer mecanismo para la pelota.
    * - Versión 3
      - 29/9/26 - 1/10/26
      - Configuración tipo tanque con disposición simétrica de las ruedas para poder girar exactamente sobre su propio eje.


Evolución del diseño
*********************

.. toctree::
    :caption: Evolución del diseño
    :maxdepth: 1

    Prototipo inicial <Evolucion/Prototipo>
    Versión 1 <Evolucion/Version1>
    Versión 2 <Evolucion/Version2>
    Versión 3 <Evolucion/Version3>

.. toctree::
    :caption: Mecanismos
    :maxdepth: 1

    Mecanismo para recoger la pelota <Mecanismos/Pelota>
    Sistema de orugas (descartado) <Mecanismos/Orugas>

