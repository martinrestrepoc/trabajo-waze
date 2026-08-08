# Roadmap: enrutamiento A* con OpenStreetMap

## 1. Objetivo

Desarrollar en `Waze_IA.ipynb` un método propio de búsqueda de rutas basado en A* para encontrar una ruta entre dos puntos de La Estrella, Antioquia.

La función objetivo será **minimizar el tiempo estimado total de viaje en automóvil**. No se utilizarán los algoritmos de rutas de NetworkX, como `nx.shortest_path`, `nx.astar_path` o `nx.dijkstra_path`.

Sí se podrán utilizar OSMnx y NetworkX para:

- descargar y almacenar la red vial;
- consultar nodos, vecinos y aristas;
- encontrar el nodo vial más cercano a una coordenada;
- acceder a los atributos de OpenStreetMap;
- visualizar el grafo y la ruta final.

Los dos lugares fijos de la demostración serán:

- origen: unidad o edificio de apartamentos Pitriza;
- destino: corregimiento La Tablaza, La Estrella.

Antes de ejecutar la búsqueda se verificarán visualmente las ubicaciones obtenidas por el geocodificador. Si alguna ubicación no es reconocida correctamente, se usarán coordenadas fijas verificadas.

## 2. Decisión de diseño

Se creará una clase especializada, por ejemplo `OSMRouteMap`, inspirada en la clase `RouteMap` del cuaderno `IA_Busquedas_Unificado.ipynb`.

No se reutilizará directamente `Tree.aStar`, porque una red vial tiene miles de nodos y puede llegar al mismo nodo mediante rutas diferentes. La implementación nueva conservará la lógica estudiada en el cuaderno, pero añadirá:

- registro del mejor tiempo conocido para cada nodo;
- actualización de un nodo cuando se encuentre un camino más rápido;
- control global de los nodos explorados;
- reconstrucción de la ruta mediante predecesores;
- manejo de las diferentes aristas de un `MultiDiGraph`.

Cada estado representará el identificador de un nodo de OpenStreetMap. Los operadores serán las vías dirigidas que salen de ese nodo.

## 3. Función de costo y heurística

### 3.1 Costo acumulado `g(n)`

`g(n)` representará los segundos estimados transcurridos desde el origen hasta el nodo actual.

El costo de recorrer una arista comenzará con su atributo `travel_time`, calculado por OSMnx a partir de la longitud y la velocidad estimada. Después se podrán incorporar ajustes que también se expresen como tiempo o como factores sobre el tiempo:

```text
costo_tramo =
    travel_time
    × factor_tipo_vía
    × factor_superficie
    × factor_pendiente
    + demora_semáforo
```

Todos los factores tendrán un valor predeterminado de `1.0`. Las penalizaciones nunca serán negativas.

No se sumarán directamente distancia, velocidad y tiempo como tres costos independientes, porque `travel_time` ya se obtiene usando distancia y velocidad. Hacerlo contaría varias veces la misma información y mezclaría metros, kilómetros por hora y segundos.

### 3.2 Heurística `h(n)`

`h(n)` estimará los segundos que faltan para llegar desde el nodo actual hasta el destino:

```text
h(n) = distancia_recta_en_metros × 3.6 / velocidad_máxima_en_km_h
```

La multiplicación por `3.6` convierte el resultado a segundos. La distancia en línea recta se calculará con las coordenadas de los nodos. La velocidad de referencia será igual o superior a la mayor velocidad asignada a las vías del grafo.

Esta heurística es sencilla, rápida y coherente con el objetivo de minimizar el tiempo. Los semáforos, la superficie y el tipo de vía no se incluirán directamente en `h(n)`, porque no se conoce de antemano cuáles encontrará la ruta restante. Esos atributos modificarán el costo real `g(n)` cuando se recorran las aristas.

### 3.3 Prioridad de A*

La cola de prioridad ordenará los nodos mediante:

```text
f(n) = g(n) + h(n)
```

Todas las partes de `f(n)` estarán expresadas en segundos.

## 4. Selección de atributos

| Atributo | Uso | Complejidad | Decisión inicial |
|---|---|---:|---|
| `length` | Calcular tiempo y distancia final | Baja | Incluir |
| `speed_kph` | Calcular el tiempo de cada tramo | Baja | Incluir |
| `travel_time` | Costo temporal base | Baja | Incluir |
| `highway` | Ajustar el tiempo según el tipo de vía | Baja | Incluir |
| Semáforos | Agregar una demora fija al atravesarlos | Media | Incluir |
| `surface` | Penalizar superficies que reduzcan la velocidad | Media | Incluir si hay cobertura suficiente |
| Pendiente | Ajustar el tiempo en subidas pronunciadas | Alta | Incorporar en una fase opcional |

La primera versión funcional utilizará solamente `travel_time` para `g(n)` y el tiempo en línea recta para `h(n)`. Los demás atributos se incorporarán uno por uno después de comprobar que el algoritmo básico funciona.

## 5. Roadmap de implementación

### Fase 1. Preparar el notebook y el mapa

- [ ] Agregar a `Waze_IA.ipynb` el título, objetivo y explicación del problema.
- [ ] Importar `osmnx`, `heapq`, `math`, `time`, `pandas` y las herramientas de visualización necesarias.
- [ ] Descargar la red vial con `graph_from_place("La Estrella, Antioquia, Colombia", network_type="drive")`.
- [ ] Mantener el sentido de circulación de las vías.
- [ ] Dibujar el grafo para comprobar que la descarga fue correcta.

**Resultado:** grafo vial dirigido de La Estrella disponible en el notebook.

### Fase 2. Definir origen y destino

- [ ] Geocodificar Pitriza y La Tablaza.
- [ ] Mostrar las direcciones y coordenadas encontradas.
- [ ] Confirmar visualmente que corresponden a los lugares esperados.
- [ ] Guardar coordenadas manuales como respaldo.
- [ ] Convertir las coordenadas a nodos mediante `ox.distance.nearest_nodes`.
- [ ] Dibujar los nodos de origen y destino sobre el mapa.

**Resultado:** identificadores de los nodos inicial y final verificados.

### Fase 3. Preparar y auditar los datos viales

- [ ] Definir velocidades predeterminadas por tipo de vía para los casos sin `maxspeed`.
- [ ] Calcular `speed_kph` con `ox.add_edge_speeds`.
- [ ] Calcular `travel_time` con `ox.add_edge_travel_times`.
- [ ] Contar cuántas aristas contienen `length`, `speed_kph`, `travel_time`, `highway` y `surface`.
- [ ] Contar los nodos identificados como `highway=traffic_signals`.
- [ ] Revisar los valores diferentes y los datos faltantes de cada atributo.
- [ ] Documentar las reglas utilizadas para reemplazar valores ausentes.

**Resultado:** tabla de cobertura que permita decidir qué atributos de OSM son utilizables en La Estrella.

### Fase 4. Crear la clase `OSMRouteMap`

La clase tendrá, como mínimo, la siguiente interfaz:

```python
class OSMRouteMap:
    def __init__(self, graph, config=None):
        ...

    def neighbors(self, node_id):
        ...

    def edge_travel_cost(self, u, v, key, edge_data):
        ...

    def heuristic(self, node_id, destination_id):
        ...

    def a_star(self, start_id, destination_id):
        ...

    def reconstruct_route(self, came_from, destination_id):
        ...
```

- [ ] Guardar el grafo y la configuración de penalizaciones.
- [ ] Obtener únicamente vecinos alcanzables respetando el sentido de las vías.
- [ ] Evaluar todas las aristas paralelas entre dos nodos.
- [ ] Devolver el costo de cada tramo en segundos.
- [ ] Calcular la heurística temporal hasta el destino.

**Resultado:** clase capaz de consultar la red y calcular costos sin utilizar métodos de rutas de NetworkX.

### Fase 5. Implementar A* desde cero

- [ ] Crear una cola de prioridad mediante `heapq`.
- [ ] Inicializar el tiempo del origen en cero.
- [ ] Mantener un diccionario `g_score` con el menor tiempo conocido para cada nodo.
- [ ] Mantener un diccionario `came_from` con el nodo y la arista predecesores.
- [ ] Extraer siempre el estado con menor `f(n)`.
- [ ] Descartar entradas desactualizadas de la cola.
- [ ] Actualizar un vecino cuando se encuentre un tiempo menor.
- [ ] Terminar cuando el destino sea extraído de la cola.
- [ ] Reconstruir la secuencia de nodos y las claves de las aristas.
- [ ] Reportar claramente si no existe una ruta.

**Resultado:** ruta válida producida completamente por nuestra implementación.

### Fase 6. Validar la versión mínima

La primera configuración tendrá:

```text
g(n) = suma de travel_time
h(n) = tiempo en línea recta hasta el destino
```

- [ ] Verificar que la ruta empiece en el origen y termine en el destino.
- [ ] Verificar que cada par consecutivo de nodos tenga una arista dirigida válida.
- [ ] Verificar que todos los costos sean finitos y no negativos.
- [ ] Verificar que `h(destino) = 0`.
- [ ] Ejecutar el mismo método con `h(n) = 0` como prueba de costo uniforme propia.
- [ ] Comparar los tiempos encontrados por ambas configuraciones.
- [ ] Registrar cantidad de nodos explorados y tiempo de ejecución.

No se utilizará `nx.shortest_path` como comparación ni como mecanismo de validación.

**Resultado:** evidencia de que la implementación básica funciona correctamente.

### Fase 7. Incorporar los atributos progresivamente

#### Iteración 7.1: semáforos

- [ ] Detectar si el nodo de llegada de una arista es un semáforo.
- [ ] Agregar una demora configurable, por ejemplo 15 o 20 segundos.
- [ ] Comparar la ruta antes y después de la penalización.

#### Iteración 7.2: tipo de vía

- [ ] Crear un diccionario de factores por `highway`.
- [ ] Tratar correctamente valores únicos, listas y datos faltantes.
- [ ] Aplicar únicamente penalizaciones justificables y moderadas.

#### Iteración 7.3: superficie

- [ ] Incorporar `surface` solamente si su cobertura en el mapa es suficiente.
- [ ] Agrupar superficies en categorías simples: pavimentada, irregular y desconocida.
- [ ] Evitar crear reglas para demasiados valores particulares.

#### Iteración 7.4: pendiente opcional

- [ ] Conseguir elevaciones para los nodos mediante una fuente compatible con OSMnx.
- [ ] Calcular `grade` para las aristas.
- [ ] Analizar si la pendiente cambia significativamente el tiempo estimado en automóvil.
- [ ] Incorporarla solo si mejora el modelo sin volverlo innecesariamente complejo.

**Resultado:** modelo temporal configurable enriquecido con los atributos que realmente estén disponibles.

### Fase 8. Visualizar y explicar la ruta

- [ ] Dibujar la ruta final sobre el grafo con OSMnx.
- [ ] Diferenciar visualmente origen y destino.
- [ ] Crear una tabla con los tramos seleccionados y sus atributos.
- [ ] Calcular distancia total en metros o kilómetros.
- [ ] Calcular tiempo base y tiempo ajustado.
- [ ] Contar semáforos atravesados.
- [ ] Resumir los tipos de vía y superficies utilizados.
- [ ] Mostrar el número de nodos explorados por A*.

**Resultado:** visualización y resumen interpretables de la ruta Pitriza–La Tablaza.

### Fase 9. Experimentos finales

Se compararán configuraciones construidas con el mismo A* propio:

1. solo tiempo de viaje;
2. tiempo más semáforos;
3. tiempo, semáforos y tipo de vía;
4. configuración completa con los atributos que tengan buena cobertura.

Para cada configuración se registrará:

- ruta encontrada;
- tiempo base estimado;
- tiempo ajustado por la función de costo;
- distancia total;
- semáforos atravesados;
- nodos explorados;
- duración de la búsqueda.

**Resultado:** explicación de cómo cada atributo modifica la ruta y el desempeño de A*.

## 6. Criterios de finalización

El trabajo estará terminado cuando:

- `Waze_IA.ipynb` pueda ejecutarse en orden desde el inicio hasta el final;
- descargue o cargue la red vial de La Estrella;
- identifique correctamente Pitriza y La Tablaza;
- encuentre una ruta mediante un A* implementado por nosotros;
- no llame a algoritmos de rutas de NetworkX;
- mida todos los costos en segundos;
- muestre la ruta y sus métricas principales;
- permita cambiar penalizaciones sin modificar el algoritmo;
- explique claramente la diferencia entre `g(n)`, `h(n)` y `f(n)`;
- documente las limitaciones de los datos faltantes de OpenStreetMap.

## 7. Alcance recomendado para la primera entrega

La primera entrega debería incluir:

- distancia;
- velocidad estimada;
- tiempo estimado;
- tipo de vía;
- semáforos;
- superficie únicamente si existe información suficiente.

La pendiente se conservará como ampliación opcional. El objetivo principal es demostrar correctamente la implementación propia de A* y la minimización del tiempo estimado, antes de aumentar la complejidad del modelo.

## 8. Limitaciones conocidas

- `travel_time` representa tiempo de circulación libre y no tráfico en tiempo real.
- Las etiquetas de semáforos y superficies pueden estar incompletas.
- Las velocidades faltantes serán estimadas según el tipo de vía.
- Las penalizaciones representan supuestos del modelo y deberán explicarse.
- La calidad de la ruta dependerá de la calidad y actualidad de los datos de OpenStreetMap.

## 9. Referencias técnicas

- [OSMnx 2.1.1: referencia de usuario](https://osmnx.readthedocs.io/en/stable/user-reference.html)
- [OpenStreetMap: clasificación de vías](https://wiki.openstreetmap.org/wiki/Highways)
- [OpenStreetMap: etiqueta `surface`](https://wiki.openstreetmap.org/wiki/Surface)
