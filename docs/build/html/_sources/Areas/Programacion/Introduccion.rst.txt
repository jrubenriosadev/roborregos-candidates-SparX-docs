#####################
Área de Programación
#####################

Dentro de la programación de RIB-EYE se utilizaron distintas herramientas para la resolución de los retos.
Las siguientes explicaciones de código son asumiendo que ya se tienen ciertos conocimientos previos de C++ y programación de 
microcontroladores como Arduino.

Herramientas utilizadas
************************

1. **PlatformIO:** Este es un entorno de desarrollo integrado (IDE) que te permite escribir, compilar y cargar código en prácticamente cualquier
microcontrolador, ya que PlatformIO es compatible con muchísimos microcontroladores y placas. En nuestro caso, utilizamos un microcontrolador ESP32.

2. **C++:** Este es un lenguaje de programación que tiene un fuerte enfoque en el control de bajo nivel, lo que nos permite tener una manipulación
más directa del hardware y de los recursos del microcontrolador, además de ser un lenguaje bastante eficiente en cuanto a su velocidad y tiempo de ejecución.


Resolución de retos
********************

.. toctree::
   :caption: Componentes
   :maxdepth: 2

   Configuración de componentes <ComponentesProgra>

.. toctree::
   :caption: PID
   :maxdepth: 2

   Introducción <PID/IntroduccionPID>

.. toctree::
   :caption: Contenedor y Máquina de Estados
   :maxdepth: 2

   Introducción al contenedor (RobotContainer) <Contenedor/Introduccion>
   Introducción a la máquina de estados (StateMachine) <MaquinaEstados/Introduccion>

.. toctree::
   :caption: Subsistemas
   :maxdepth: 2

   Introducción a los subsistemas <Subsistemas/Introduccion>


.. toctree::
   :caption: Algoritmo de resolución de maze
   :maxdepth: 1

   Planteamiento de resolución de maze<Algoritmo/Planteamiento>