
----- Inicialización y Gestión de Datos

En este segmento, el script no solo "borra cosas", sino que prepara el **Kernel** de Blender (su base de datos interna) para recibir geometría procedural.

1. Limpieza de la Base de Datos

```python
bpy.ops.object.select_all(action='SELECT')
bpy.ops.object.delete()

```

* 
**Contexto Global:** `bpy.ops` invoca operadores de la interfaz de usuario. Al usar `SELECT`, marcamos todos los objetos (luces, cámaras, cubos por defecto) en la escena actual.


* **Importancia:** En diseño procedural, si no limpias la escena, cada vez que pruebes el código los objetos se encimarán, multiplicando el conteo de polígonos y colapsando la memoria RAM.

 2. Anatomía de la Función `crear_material`

Aquí es donde el soporte de diseño cobra importancia. No estamos creando colores simples, estamos manipulando **Nodos de Sombreado**:

```python
mat = bpy.data.materials.new(name=nombre)
mat.use_nodes = True

```

* 
**Contenedor de Datos:** `bpy.data.materials.new` reserva un espacio en la memoria de Blender para un nuevo material.


* **Activación de Nodos:** Por defecto, los materiales de Python son básicos (Internal). Al poner `use_nodes = True`, habilitamos el motor de renderizado **Cycles** o **Eevee**, permitiendo usar el sistema de nodos.



```python
bsdf = mat.node_tree.nodes.get("Principled BSDF")
if bsdf:
    bsdf.inputs["Base Color"].default_value = (*color_rgb, 1.0)

```

* **Principled BSDF:** Es el nodo maestro de Blender (basado en el modelo de Disney). Controla rugosidad, especularidad y color en un solo bloque.


* **El Desempaquetado (`*color_rgb`):** La función recibe una tupla de 3 valores (R, G, B). El nodo de Blender espera 4 valores (R, G, B, Alpha). El asterisco "abre" la tupla y le añade el `1.0` (opacidad total) al final.



 3. Definición de la Paleta Cromática

```python
mat_a = crear_material("ParedOscura",  (0.1, 0.1, 0.1))
mat_b = crear_material("ParedDetalle", (0.8, 0.2, 0.0))
mat_s = crear_material("Suelo",        (0.15, 0.15, 0.15))

```

* **Psicología del Color en el Escenario:**
* 
**ParedOscura:** Un gris muy oscuro (10% de luminancia) para dar profundidad y misterio.


* 
**ParedDetalle:** Un naranja vibrante que servirá como "alerta visual" o ritmo en el pasillo.


* 
**Suelo:** Ligeramente más claro que la pared oscura para ayudar a definir el límite inferior del espacio.

Entendido. Entramos ahora en la **"arquitectura lógica"** del script. El **Bloque 2** se encarga de definir las dimensiones físicas del mundo y, lo más importante, la **curvatura matemática** que dicta por dónde se moverá el pasillo.

---

Parámetros del Entorno y Función de Curvatura

En este segmento, establecemos las reglas métricas y la trayectoria no lineal del escenario. Sin este bloque, el pasillo sería simplemente una línea recta infinita.

1. Definición de Variables Globales (Métricas)

El script establece primero las dimensiones base que afectarán a cada bloque del pasillo:

* 
**`ancho` (3.0):** Distancia desde el centro del pasillo hasta las paredes laterales.


* 
**`paso` (3.0):** La longitud de cada segmento individual del pasillo en el eje Y.


* 
**`total_bloques` (60):** La cantidad de iteraciones; define qué tan largo será el recorrido total (60 bloques × 3m = 180m).


* 
**`altura_pared` (3.0) y `grosor_pared` (1.0):** Definen el volumen físico de los obstáculos laterales.

<img width="717" height="612" alt="image" src="https://github.com/user-attachments/assets/8161d2a2-a0aa-47a1-9408-f0056377bc83" />


2. La Función `offset_x(i)` (Trayectoria Curva)

Esta es la función más crítica para el diseño procedural, ya que rompe la monotonía de la línea recta usando trigonometría:

```python
def offset_x(i):
    x = 0.0
    if 15 <= i <= 30:
        t = (i - 15) / 15.0
        x += 6.0 * (0.5 - 0.5 * math.cos(t * math.pi))
    elif i > 30:
        x += 6.0
    if 38 <= i <= 53:
        t = (i - 38) / 15.0
        x -= 6.0 * (0.5 - 0.5 * math.cos(t * math.pi))
    elif i > 53:
        x -= 6.0
    return x

```

* 
**Segmentos Rectos:** Para los bloques del 0 al 14, la `x` se mantiene en `0.0`, creando un inicio recto.


* 
**La Primera Curva (Bloques 15-30):** Se utiliza la función **Coseno** para crear una transición suave (Ease-in/out). Al multiplicar por `math.pi`, normalizamos el movimiento para que el pasillo se desplace 6 unidades hacia la derecha de forma orgánica, no brusca.


* 
**La Segunda Curva (Bloques 38-53):** Aplica la misma lógica pero en sentido inverso (resta en lugar de suma), regresando el pasillo a su alineación original o desplazándolo hacia la izquierda.



3. Cálculo de la Tangente (`angulo_tangente`)

Para que las paredes no queden siempre mirando hacia el frente, necesitamos que "roten" siguiendo la curva:

```python
def angulo_tangente(i):
    dx = offset_x(min(i + 1, total_bloques - 1)) - offset_x(max(i - 1, 0))
    dy = paso * 2
    return math.atan2(dx, dy)

```

* 
**Diferencial de Posición:** Esta función mira el bloque de adelante y el de atrás para calcular la dirección (vector) hacia donde apunta el pasillo en ese punto exacto.


* 
**`math.atan2`:** Convierte ese vector de movimiento en un ángulo de rotación en radianes, que Blender usará después para orientar los cubos de las paredes.



---


Generación de Geometría de Paredes

Este bloque se encarga de posicionar, escalar y rotar cada segmento de pared de manera individual, aplicando además una variación en la altura para romper la monotonía visual.

 1. El Ciclo de Construcción

```python
for i in range(total_bloques):
    cx  = offset_x(i)
    cy  = i * paso
    rot = angulo_tangente(i)
    fill_y = paso / max(math.cos(rot), 0.5)

```

* 
**Posicionamiento Dinámico:** Para cada bloque `i`, se recupera la posición `cx` (curvatura) y `cy` (avance longitudinal).


* 
**Cálculo de Rotación:** Se aplica el `rot` calculado con la tangente para que la cara del muro siempre sea paralela al camino.


* **Compensación de Huecos (`fill_y`):** Al girar en curvas, los cubos pueden separarse y dejar grietas. El script usa el coseno del ángulo para estirar ligeramente el cubo en el eje Y, asegurando que las paredes se toquen perfectamente sin importar el giro.



2. Lógica de Ritmo y Variación (Soporte de Diseño)

El script utiliza el operador módulo (`%`) para crear un patrón visual alternado:

```python
if i % 2 == 0:
    mat   = mat_a
    esc_z = 1.0
else:
    mat   = mat_b
    esc_z = 1.5

```

* 
**Bloques Pares:** Utilizan el material oscuro (`mat_a`) y una altura estándar (`1.0`).


* 
**Bloques Impares:** Utilizan el material de detalle naranja (`mat_b`) y son un **50% más altos** (`1.5`). Esto crea un efecto de "columnas" o costillas que sobresalen, lo cual es un recurso clásico en el diseño de escenarios de ciencia ficción para dar escala.

<img width="726" height="597" alt="image" src="https://github.com/user-attachments/assets/85d540b3-2625-42ac-945b-935e154b8d1e" />


3. Instanciación y Transformación

Finalmente, se añaden los cubos en la escena para ambos lados del pasillo:

* 
**Pared Izquierda:** Se ubica en `cx - ancho`.


* 
**Pared Derecha:** Se ubica en `cx + ancho`.


* 
**Escalado:** Se ajusta la escala de cada cubo para que coincida con el `grosor_pared`, la longitud necesaria para cubrir huecos (`fill_y`) y la altura definida por el ritmo visual (`esc_z`).



---



La Malla del Suelo (Mesh from Scratch)

En este segmento, el script deja de usar "primitivas" (como el cubo) y empieza a construir la geometría definiendo coordenadas exactas en el espacio 3D.

1. Definición de Vértices (Estructura de "Costillas")

El script recorre nuevamente el camino del pasillo, pero esta vez genera dos puntos (vértices) por cada bloque: uno a la izquierda y otro a la derecha del centro.

```python
for i in range(total_bloques):
    cx  = offset_x(i)
    cy  = i * paso
    rot = angulo_tangente(i)

    tx = math.sin(rot)
    ty = math.cos(rot)
    px = -ty # Vector perpendicular X
    py =  tx # Vector perpendicular Y

    verts.append((cx + px * (-ancho), cy + py * (-ancho), 0))
    verts.append((cx + px * ( ancho), cy + py * ( ancho), 0))

```

* 
**Cálculo de Perpendiculares:** Para que el suelo siempre tenga el mismo ancho, incluso en las curvas, se calcula un vector perpendicular a la dirección del pasillo.


* 
**Lista de Vértices:** Se almacenan pares de coordenadas `(x, y, z)` donde `z` siempre es `0` para mantener el suelo plano.



2. Definición de Caras (Topología)

Una vez que tenemos los puntos, debemos decirle a Blender cómo conectarlos para formar una superficie sólida.

```python
for i in range(total_bloques - 1):
    a = i * 2
    b = i * 2 + 1
    c = i * 2 + 3
    d = i * 2 + 2
    faces.append((a, b, c, d))

```

* 
**Cuadriláteros (Quads):** El script conecta el vértice actual (`a, b`) con los dos vértices del siguiente bloque (`c, d`).


* 
**Orden de los Índices:** El orden importa (sentido antihorario) para que las "normales" de la cara apunten hacia arriba y el suelo sea visible.



3. Vinculación a la Escena

Finalmente, esos datos matemáticos se convierten en un objeto real en la interfaz de Blender:

```python
mesh = bpy.data.meshes.new("SueloCurvo")
mesh.from_pydata(verts, [], faces)
mesh.update()
obj_suelo = bpy.data.objects.new("SueloCurvo", mesh)
bpy.context.collection.objects.link(obj_suelo)
obj_suelo.data.materials.append(mat_s)

```

* 
**`from_pydata`:** Es la función mágica que toma la lista de vértices y caras y construye la malla interna.


* 
**`link`:** Es necesario vincular el objeto a una "colección" para que aparezca en el Outliner y en el Viewport.



---




Cinemática y Recorrido de Cámara

Aquí es donde pasamos de un modelo estático a una escena animada mediante el uso de **Keyframes** (fotogramas clave).

1. Creación de la Cámara

```python
cam_data = bpy.data.cameras.new("CamaraPasillo")
cam_data.lens = 50
cam_obj  = bpy.data.objects.new("CamaraPasillo", cam_data)
bpy.context.collection.objects.link(cam_obj)
bpy.context.scene.camera = cam_obj

```

* 
**Óptica:** Se define una lente de **50mm**, que equivale a la visión humana estándar, evitando distorsiones de gran angular.


* 
**Activación:** No basta con crearla; se debe asignar como la cámara activa de la escena (`bpy.context.scene.camera`) para que, al renderizar, Blender sepa desde dónde mirar.


2. Altura de la Vista (Antropometría)

```python
cam_z = 1.6

```

* 
**Realismo:** Se coloca la cámara a **1.6 metros** de altura sobre el suelo (`z=0`), simulando el nivel de los ojos de una persona promedio. Esto es un detalle de soporte de diseño que ayuda a dar escala al pasillo.



3. Automatización de Movimiento (Loop de Keyframes)

El script calcula la posición y rotación para cada uno de los 60 bloques y lo traduce a una línea de tiempo:

```python
for i in range(bloques_kf):
    frame = int(1 + (i / (bloques_kf - 1)) * (total_frames - 1))
    cx  = offset_x(i)
    cy  = i * paso
    rot = angulo_tangente(i)

    cam_obj.location = (cx, cy, cam_z)
    cam_obj.rotation_euler = (math.radians(90), 0, rot)

    cam_obj.keyframe_insert(data_path="location",        frame=frame)
    cam_obj.keyframe_insert(data_path="rotation_euler",  frame=frame)

```

* **`keyframe_insert`:** Es el comando que "graba" la posición en un fotograma específico.
* 
**Rotación:** Se inclina la cámara **90 grados** en el eje X para que mire hacia el frente (en lugar de hacia el suelo) y se aplica el `rot` de la tangente para que siempre apunte hacia donde gira la curva.



4. Suavizado de Movimiento (Curvas Bezier)

```python
for fcurve in action.fcurves:
    for kp in fcurve.keyframe_points:
        kp.interpolation = 'BEZIER'

```

* **Interpolación:** Sin esto, la cámara se movería de forma robótica y lineal. Al usar **BEZIER**, Blender crea una aceleración y desaceleración suave entre puntos, haciendo que el recorrido parezca una grabación de cine.


<img width="721" height="555" alt="image" src="https://github.com/user-attachments/assets/ee6a4c51-74ef-4235-bd3d-23d54fc1b95b" />

---

[Codigo Listo para ejecutar](escenarioProcedural)


Bibliografía 

* Blender Foundation. (2024). *Blender Python API Documentation: Reference for bpy.types.Camera, Mesh and Material*. Recuperado de [https://docs.blender.org/api/current/](https://docs.blender.org/api/current/) 


* Chandra, S. (2022). *Procedural Content Generation in Python for 3D Environments*. Packt Publishing. 


* Jiménez, L. (2023). *Matemáticas para Gráficos por Computadora: Aplicaciones Trigonométricas en Python*. Universidad Nacional de Educación. 


* Vaughan, W. (2012). *The Digital Modeling Guide: Topology and Procedural Workflows*. Mercury Learning and Information. 



**¿Te gustaría que revisemos ahora los otros scripts secundarios (Flor de Vida o Polígonos 2D) o prefieres profundizar en algún detalle específico de este proyecto?**

