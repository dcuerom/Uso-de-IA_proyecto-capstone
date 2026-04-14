# Anexo: Documentación de Uso de Inteligencia Artificial
## Prompts y Respuestas — Proyecto Capstone Analytics

**Proyecto:** Optimización Logística con Algoritmo Genético (TDVRPTW)  
**Equipo:** Ingeniería Industrial — UDD  
**Herramienta de IA utilizada:** Antigravity (Google DeepMind) — Asistente de generación de código  
**Período documentado:** Abril 2026

---

> **Nota sobre el uso de inteligencia artificial en este proyecto:**
>
> Conforme al marco académico establecido, el uso de herramientas de IA en este proyecto respondió a dos propósitos definidos:
>
> 1. **Generación de código Python:** Los integrantes del equipo actuaron como ingenieros industriales responsables de definir las especificaciones funcionales del sistema (qué debía hacer el código), mientras que la herramienta de IA se encargó de producir la implementación técnica. Cada fragmento de código generado fue revisado y validado por el equipo antes de su incorporación al proyecto.
>
> 2. **Tratamiento de datos:** La herramienta de IA asistió en el procesamiento y análisis de grandes volúmenes de datos, en todo momento bajo las instrucciones y la supervisión directa del equipo.
>
> El contenido de este anexo constituye el registro fiel de las conversaciones sostenidas con la herramienta de IA a lo largo del proyecto.

---

---

## SECCIÓN I: MODELAMIENTO MATEMÁTICO Y EVALUACIÓN DEL MODELO TDVRPTW

**Conversación de referencia:** *"Evaluating TDVRPTW Mathematical Model"*  
**Fecha:** Abril 2026  
**Propósito:** Generar el modelo matemático formal del problema TDVRPTW y su integración con el evaluador PyMoo.

---

### Prompt 1.1 — Solicitud de implementación del modelo híbrido Pyomo + PyMoo

**Contexto previo:** El equipo disponía de una estructura Pyomo (`modelo/modelo.py`) y requería incorporar un evaluador de alto rendimiento para el Algoritmo Genético.

**Prompt del usuario:**

> Necesito que implementes el modelo de optimización híbrido integrando Pyomo con PyMoo. El modelo debe:
> - Mantener la estructura formal de Pyomo como referencia algebraica del negocio
> - Crear un nuevo archivo `modelo/pymoo_problem.py` que implemente `TDVRPTWProblem(Problem)`
> - El evaluador debe usar `tau_ij_vec()` para vectorizar los cálculos de tiempo de viaje
> - Las restricciones de capacidad y ventanas de tiempo deben implementarse como hard constraints en `out["G"]`
> - La función objetivo `out["F"]` debe minimizar la distancia total (en metros) × factor S

**Respuesta del asistente de IA:**

El asistente presentó el siguiente plan de implementación:

**Plan de Implementación: Integración Híbrida Pyomo + PyMoo**

**1. Dualidad del Planteamiento (`modelo/`)**

- **Mantener Pyomo:** El archivo `modelo/modelo.py` conserva su estructura formal `ConcreteModel`, cumpliendo la función de esqueleto algebraico de referencia del negocio.
- **[NUEVO] `modelo/pymoo_problem.py`:**
  - Se implementó la clase `TDVRPTWProblem(Problem)`. Se optó por `Problem` (vectorizado) en vez de `ElementwiseProblem`, con el objetivo de aprovechar el procesamiento matricial de `tau_ij_vec()`.
  - **Uso de `tau_ij_vec`:** El método `_evaluate(self, x, out)` iteró sobre los tramos del recorrido, procesando a toda la población (cientos de individuos) en paralelo, matriz a matriz.
  - **Restricciones Hard:**
    - Las violaciones de capacidad o de ventana de tiempo (llegada posterior al límite `b`) se registraron como penalizaciones físicas en `out["G"]`, forzando al algoritmo a rechazar individuos infactibles mediante las reglas de dominancia de Deb.
    - `out["F"]` se reservó exclusivamente para la sumatoria minimizada de la distancia en metros multiplicada por el factor S.

**2. Refactorización en `algoritmo/`**

- **[MODIFY] `algoritmo/genetic_algorithm.py`:** Se orquestó el algoritmo con la librería oficial `from pymoo.algorithms.soo.nonconvex.ga import GA`, inyectando la matriz de clientes y la configuración a `TDVRPTWProblem`, con cruces generados mediante `OrderCrossover()` nativo de PyMoo.
- **[MODIFY] `algoritmo/tabu_search.py`:** Se mantuvo el ciclo nativo de búsqueda tabú. Para evaluar la calidad de cada vecino, se encapsuló la llamada al simulador, inyectando el cromosoma al vector de `TDVRPTWProblem().evaluate([cromosoma])`, garantizando consistencia 1:1 en las penalizaciones entre ambos métodos.

**3. Generación de Reportes**

Ambos procesos volcaron los resultados de sus mejores cromosomas factibles en `resultados/rutas/` en un formato legible.

---

### Prompt 1.2 — Solicitud de penalización por espera, salida flexible y exportación CSV

**Contexto previo:** Tras las primeras ejecuciones, el equipo detectó que determinados vehículos acumulaban hasta 463 minutos de espera (Vehículo 16) como consecuencia de una hora de salida fija desde el depósito, lo que evidenciaba una ineficiencia operacional significativa.

**Prompt del usuario:**

> Quiero implementar 3 cambios en el modelo:
> 1. **Penalización por tiempo de espera:** Si un camión llega antes de que abra la ventana del cliente, ese tiempo de espera debe penalizar la función objetivo con un parámetro configurable `alpha_espera`.
> 2. **Salida flexible desde el depósito:** En vez de forzar a todos los camiones K11 a salir exactamente a las 09:00, quiero que la hora de salida sea flexible dentro del intervalo del turno, usando una heurística JIT (Just-In-Time).
> 3. **Exportación a CSV:** Los reportes deben dejar de generarse en `.md` y pasar a dos archivos `.csv` separados: uno de resumen de camiones y otro de detalle de paradas.

**Respuesta del asistente de IA:**

**Cambio 1 — Nueva Función Objetivo con Penalización por Espera:**

Se redefinió la formulación matemática de la función objetivo de la siguiente manera:

```
min: sum_ijk X_(i,t),j,k * C_ij  +  alpha_w * sum_ik W_ik
```

Donde `W_ik = max(0, a_i - ts_ik)` representa el tiempo de espera del vehículo `k` en el cliente `i`, y `alpha_w` es un peso de penalización configurable.

Los cambios implementados en `pymoo_problem.py` consistieron en agregar el parámetro `alpha_espera` al constructor (valor por defecto: `1.0`), acumular el tiempo total de espera dentro del bucle `_evaluate` y sumarlo a `out["F"]` de la siguiente forma:

```python
F = (dist_total * S) + costo_vehiculos + (alpha_espera * espera_total)
```

**Cambio 2 — Salida Flexible (Restricciones 10' a 13'):**

Las restricciones de salida fija se convirtieron en desigualdades de intervalo:

| Turno | Intervalo |
|-------|-----------|
| K11 | 540 ≤ ts_0,k + s_0 ≤ 900 (09:00 – 15:00) |
| K12 | 900 ≤ ts_0,k + s_0 ≤ 1260 (15:00 – 21:00) |
| K21 | 660 ≤ ts_0,k + s_0 ≤ 1020 (11:00 – 17:00) |
| K22 | 1020 ≤ ts_0,k + s_0 ≤ 1380 (17:00 – 23:00) |

Se implementó la función `_calcular_salida_optima(ruta_clientes, turno_intervalo)`, que identificó la ventana de apertura más temprana entre los clientes asignados al turno, sustrajo el tiempo de viaje estimado desde el depósito hasta el primer cliente, y delimitó el resultado al intervalo `[lb, ub]` del turno correspondiente.

**Cambio 3 — Exportación a dos archivos CSV:**

Se reemplazó la generación de Markdown en `gestor.py` por la construcción de dos DataFrames de Pandas, exportados como `resumen_camiones_*.csv` y `detalle_paradas_*.csv`.

**Archivos modificados:**

| Archivo | Tipo de cambio |
|---------|---------------|
| `modelo/modelo.md` | MODIFY — Nueva función objetivo y restricciones 10' a 13' |
| `modelo/pymoo_problem.py` | MODIFY — alpha_espera, intervalos de turno, salida óptima |
| `algoritmo/genetic_algorithm.py` | MODIFY — Transmisión de alpha_espera al constructor |
| `gestion_flota/gestor.py` | MODIFY — Reemplazo de generación Markdown por DataFrames y CSV |

**Validación realizada:**

```bash
python3 -m py_compile modelo/pymoo_problem.py       # Sin errores
python3 -m py_compile algoritmo/genetic_algorithm.py # Sin errores
python3 -m py_compile gestion_flota/gestor.py        # Sin errores
```

---

### Resumen de lo construido en esta sesión

A través de esta sesión, el asistente de IA generó e implementó los siguientes componentes:

1. La clase `TDVRPTWProblem`, heredando de `ElementwiseProblem` (PyMoo), con vectorización mediante `tau_ij_vec()` y restricciones hard configurables.
2. La integración del GA de PyMoo con `OrderCrossover`, mutación por inversión y muestreo permutacional.
3. La penalización por tiempo de espera en la función objetivo y la lógica de salida flexible JIT desde el depósito.
4. El sistema de exportación de reportes a dos archivos CSV organizados en resumen y detalle de paradas.

---

---

## SECCIÓN II: ANÁLISIS DE OPERACIONES DE FLOTA — ASIGNACIÓN DINÁMICA DE TURNOS

**Conversación de referencia:** *"Analyzing Fleet Management Operations"*  
**Fecha:** 13 de Abril, 2026  
**Propósito:** Eliminar la asignación forzada del turno de mañana (K11) a rutas con entregas en horario vespertino, mediante la implementación de una selección dinámica del turno basada en las ventanas de tiempo del primer cliente de cada ruta.

---

### Prompt 2.1 — Corrección de la asignación de turno forzada

**Contexto previo:** Al revisar los resultados del algoritmo genético, el equipo constató que el sistema asignaba el turno K11 (mañana, salida 09:00) a rutas cuyo primer cliente registraba ventanas de tiempo vespertinas (por ejemplo, 14:00 hrs), lo que generaba tiempos de espera excesivos e incoherencia con los horarios operacionales reales.

**Prompt del usuario:**

> El algoritmo está asignando el turno K11 (mañana, salida 09:00) a rutas que deberían pertenecer al turno de tarde (K21 o K22). El problema es que la lógica actual siempre inicia en K11 independientemente de cuándo abren los clientes. Necesito que el algoritmo determine dinámicamente el turno óptimo de inicio (K11, K12, K21, o K22) mirando la ventana de tiempo del primer cliente de cada ruta para minimizar el tiempo ocioso y que la asignación sea coherente con los horarios reales de entrega.

**Respuesta del asistente de IA:**

El asistente identificó que en `pymoo_problem.py` la evaluación asignaba siempre el primer turno disponible de forma secuencial, sin considerar el horario de atención del primer cliente. La solución propuesta consistió en la siguiente heurística de selección dinámica:

```python
def _seleccionar_turno_inicial(self, primer_cliente_ventana_apertura):
    """
    Selecciona el turno de despacho más adecuado en función de
    la ventana de apertura del primer cliente de la ruta.
    Retorna (turno_tipo, ruta_idx) que minimiza el tiempo de espera.
    """
    mejor_turno = None
    menor_espera = float('inf')

    for tipo_idx, rutas in enumerate(self.plantillas_turnos):
        for ruta_idx, (lb, ub) in enumerate(rutas):
            t_salida = max(lb, primer_cliente_ventana_apertura - t_viaje_estimado)
            t_salida = min(t_salida, ub)
            espera = max(0, primer_cliente_ventana_apertura - (t_salida + t_viaje_estimado))
            if espera < menor_espera:
                menor_espera = espera
                mejor_turno = (tipo_idx, ruta_idx)

    return mejor_turno
```

Con este cambio, la lógica de evaluación seleccionó dinámicamente el turno que minimizó el tiempo de espera inicial, asignando de forma automática los turnos vespertinos (K21/K22) a las rutas cuyos clientes operaban en horario de tarde.

---

---

## SECCIÓN III: OPTIMIZACIÓN HÍBRIDA Y PARALELIZACIÓN

**Conversación de referencia:** *"Hybrid Optimization And Parallelization"*  
**Fecha:** Abril 2026  
**Propósito:** Migrar de un Algoritmo Genético con población completamente aleatoria a un paradigma híbrido que utilizara la solución del Algoritmo de Ahorros de Clarke-Wright como semilla inicial, y paralelizar la optimización de clústeres mediante multiprocesamiento.

---

### Prompt 3.1 — Implementación del paradigma híbrido Savings + GA con paralelización

**Contexto previo:** El equipo observó que el Algoritmo Genético, al partir de poblaciones completamente aleatorias, exhibía tiempos de convergencia elevados y en ocasiones no encontraba soluciones factibles dentro del número de generaciones configurado. Se buscaba una estrategia constructiva que proporcionara un punto de partida de mayor calidad.

**Prompt del usuario:**

> Quiero mejorar el rendimiento del GA implementando un paradigma híbrido de dos fases:
> 1. **Fase Constructiva:** Implementar el Algoritmo de Ahorros de Clarke-Wright adaptado al TDVRPTW que genere una solución inicial factible respetando capacidades de peso, volumen y duración máxima de ruta.
> 2. **Fase de Mejora:** Usar esa solución como la "semilla" (primer individuo) de la población inicial del GA.
> 3. **Paralelización:** Reemplazar el bucle secuencial de clústeres por procesamiento paralelo usando todos los cores disponibles de la CPU con `concurrent.futures.ProcessPoolExecutor`.
>
> Para el algoritmo de ahorros, calcula `s(i,j) = d(depot,i) + d(depot,j) - d(i,j)` para todo par de clientes, ordena descendente y fusiona rutas respetando capacidades.

**Respuesta del asistente de IA:**

Antes de proceder con la implementación, el asistente planteó dos consideraciones para revisión del equipo:

> **Punto de revisión requerido:** El Algoritmo de Ahorros producirá una solución constructiva que respeta las restricciones de capacidad y duración máxima de ruta. Las ventanas de tiempo se verificarán pero no se validarán como hard-constraint en la fase de savings; esa responsabilidad recae sobre el GA en la fase posterior. ¿Es aceptable este enfoque?

> **Punto de revisión requerido:** Para la paralelización se empleará `ProcessPoolExecutor` (multiprocesamiento real, independiente del GIL de Python). La cantidad de workers será `os.cpu_count()` (todos los núcleos disponibles). ¿Se requiere limitar la cantidad máxima de workers?

Una vez confirmados estos puntos por el equipo, se implementaron los siguientes módulos:

**`algoritmo/savings.py` (NUEVO) — Clarke-Wright adaptado al TDVRPTW:**

```python
def clarke_wright_savings(df_cluster, matriz_dist, depot_id,
                          cap_vol_cm3, cap_peso_g, d_max_min):
    """
    Algoritmo de Ahorros de Clarke-Wright adaptado al TDVRPTW.

    1. Calcula la matriz de savings: s(i,j) = d(depot,i) + d(depot,j) - d(i,j)
    2. Ordena los pares por savings en orden descendente
    3. Fusiona rutas de forma iterativa, respetando:
       - Capacidad de volumen (cap_vol_cm3)
       - Capacidad de peso (cap_peso_g)
       - Duración máxima de ruta (d_max_min)
       - Solo fusiona extremos de ruta (primer o último cliente)
    4. Retorna la permutación resultante para ser inyectada al GA
    """
```

**`algoritmo/genetic_algorithm.py` — Muestreo con semilla del algoritmo de ahorros:**

```python
class SavingsSeededSampling(Sampling):
    def __init__(self, savings_permutation):
        super().__init__()
        self.savings_perm = savings_permutation

    def _do(self, problem, n_samples, **kwargs):
        X = np.full((n_samples, problem.n_var), 0, dtype=int)
        X[0] = self.savings_perm       # Primer individuo: solución del algoritmo de ahorros
        for i in range(1, n_samples):  # Individuos restantes: permutaciones aleatorias
            X[i] = np.random.permutation(problem.n_var)
        return X

# Paralelización con ProcessPoolExecutor
with ProcessPoolExecutor(max_workers=os.cpu_count()) as executor:
    futures = {
        executor.submit(resolver_cluster, cluster_id, df_c, params): cluster_id
        for cluster_id, df_c in clusters.items()
    }
    for future in as_completed(futures):
        resultados.append(future.result())
```

**Resultados verificados en prueba (muestra de 20 pedidos):**

| KPI | Valor |
|-----|-------|
| Función Objetivo Total | 211.433,41 |
| Porcentaje de Entregas a Tiempo | 100,0% |
| Distancia Total | 224,53 km |
| Vehículos Utilizados | 4 |

**Archivos creados o modificados:**

| Archivo | Acción |
|---------|--------|
| `algoritmo/savings.py` | NUEVO — Implementación Clarke-Wright |
| `algoritmo/genetic_algorithm.py` | MODIFICADO — Semilla savings y paralelismo multi-núcleo |
| `PIPELINE_GLOBAL.md` | MODIFICADO — Documentación del nuevo paradigma |

---

---

## SECCIÓN IV: RELAJACIÓN DE RESTRICCIONES DE VENTANA DE TIEMPO

**Conversación de referencia:** *"Relaxing Time Window Constraints"*  
**Fecha:** 8 de Abril, 2026  
**Propósito:** Implementar una tolerancia de 30 minutos en el cierre de las ventanas de tiempo de los clientes, y corregir el problema de Shift Spanning mediante la transición de una restricción de tiempo de conducción a una de duración total del turno.

---

### Prompt 4.1 — Relajación de 30 minutos en las ventanas de tiempo

**Contexto previo:** Durante las ejecuciones del algoritmo con datos reales, se observó que numerosas entregas eran registradas como violaciones de ventana de tiempo por márgenes reducidos (entre 1 y 10 minutos). El equipo concluyó que una tolerancia de 30 minutos resultaba operacionalmente razonable y consistente con los márgenes habituales de operación logística.

**Prompt del usuario:**

> Necesito relajar las ventanas de tiempo de los clientes en 30 minutos adicionales en el cierre. Es decir, si la ventana de un cliente es [09:00, 10:00], ahora se debe considerar como [09:00, 10:30]. Esto debe:
> 1. Actualizarse en la documentación del modelo (`modelo.md`) en la Restricción 7
> 2. Implementarse en el evaluador `pymoo_problem.py` modificando el cálculo de `t_violacion`
> 3. Reflejarse en el reporte de diagnóstico de `genetic_algorithm.py` mostrando claramente la ventana relajada

**Respuesta del asistente de IA:**

**`modelo/modelo.md` — Restricción 7 actualizada:**

La documentación matemática fue modificada para incluir explícitamente el término de tolerancia:

```
Restricción 7: ts_ik + s_i <= b_i + 30    para todo i en I, k en K
```

**`modelo/pymoo_problem.py` — Evaluación con tolerancia de 30 minutos:**

```python
# Antes de la modificación
if t_inicio_servicio > b_p:
    t_violacion = t_inicio_servicio - b_p

# Después de la modificación
if t_inicio_servicio > (b_p + 30.0):
    t_violacion = t_inicio_servicio - (b_p + 30.0)
```

El diccionario `detalle_nodos` fue enriquecido con la clave `b_ventana_relaxed = b_p + 30.0` para mantener la trazabilidad de la relajación en los reportes.

**`algoritmo/genetic_algorithm.py` — Diagnóstico mejorado:**

```python
# Antes de la modificación
print(f"        - {nid}: Servicio a las {h_lleg} | Cierre ventana: {h_cierre} | Exceso: {viola:.0f} min")

# Después de la modificación
print(f"        - {nid}: Servicio a las {h_lleg} | Cierre (+30m): {h_cierre_rel} | Exceso: {viola:.0f} min")
```

---

### Prompt 4.2 — Corrección del problema de Shift Spanning

**Contexto previo:** El equipo detectó un comportamiento no deseado denominado internamente "shift spanning": vehículos clasificados como K11 (turno de mañana) realizaban entregas en horario vespertino, debido a que la restricción de control únicamente medía el tiempo de conducción (desplazamiento entre nodos) y excluía los períodos de espera frente a los clientes.

**Prompt del usuario:**

> El algoritmo tiene un problema que llamo "shift spanning": un camión de turno K11 (09:00) puede estar esperando fuera de un cliente hasta las 15:00 y el sistema lo sigue catalogando como K11 porque solo mide el tiempo de conducción, no el tiempo total del turno.
>
> Necesito cambiar la restricción de control de duración de "tiempo máximo de conducción" a "duración máxima del turno completo" (incluyendo tiempos de espera). Si la duración total (desde la salida hasta el regreso estimado al depósito) supera los 300 minutos (5 horas), el vehículo debe cerrar esa ruta y comenzar una nueva con el siguiente turno disponible.

**Respuesta del asistente de IA:**

El asistente explicó la causa raíz del comportamiento:

> En `pymoo_problem.py`, la condición de control emplea `tiempo_conduccion_estimado`, que acumula exclusivamente los tiempos de desplazamiento entre nodos. Un vehículo puede salir a las 09:00, desplazarse 10 minutos hasta la zona de entrega, permanecer en espera durante 5 horas hasta que abra la ventana del cliente, atenderlo y regresar al depósito a las 15:00. Para la lógica de ese momento, dicho vehículo solo había "conducido" 20 minutos, lo que no superaba el umbral de 300 minutos, y el sistema lo continuaba clasificando como K11.

**Cambio implementado en `pymoo_problem.py`:**

```python
# Antes: solo medía el tiempo de conducción
if (vol_actual + vol_p > self.cap_vol_cm3) or \
   (tiempo_conduccion_estimado > self.d_max_min):
    # se cerraba la ruta y se abría una nueva

# Después: mide la duración total del turno (incluyendo esperas)
t_retorno_estimado = t_fin_tent + t_viaj_ret_tent
duracion_turno_estimada = t_retorno_estimado - cam_t_salida

if (vol_actual + vol_p > self.cap_vol) or \
   (duracion_turno_estimada > self.d_max_min):
    # se cierra la ruta y se abre una nueva
```

**Actualización en `modelo.md` — Restricción 14:**

La restricción fue reformulada para expresar que la diferencia entre el momento de regreso al depósito (incluyendo esperas) y la hora de salida no puede superar `d_max`:

```
Restricción 14 (actualizada):
ts_fin_k - ts_inicio_k <= d_max    para todo k en K
(donde ts_fin_k incorpora el tiempo de regreso al depósito con esperas acumuladas)
```

El asistente advirtió proactivamente sobre el impacto del cambio:

> Si el umbral `d_max_min` = 300 minutos se aplica de forma estricta sobre la duración total del turno, el algoritmo generará rutas más fragmentadas y realizará un mayor número de asignaciones al turno K12 u otros turnos, dado que los períodos de espera ahora se contabilizan como tiempo activo del turno. Se requiere confirmar que este comportamiento es el deseado.

El equipo confirmó que dicho comportamiento era correcto y se encontraba alineado con la realidad operacional de la empresa.

---

---

## SECCIÓN V: EXTRACCIÓN DE DATOS — PORTAL INMOBILIARIO (BODEGAS)

**Conversación de referencia:** *"Scraping Portal Inmobiliario Data"*  
**Fecha:** Abril 2026  
**Propósito:** Extraer datos de bodegas en venta en la Región Metropolitana desde Portal Inmobiliario para su análisis como potenciales centros de distribución. Uso de IA bajo la modalidad de tratamiento de datos.

---

### Prompt 5.1 — Desarrollo del scraper con Playwright

**Contexto previo:** El equipo requería recopilar información georreferenciada de bodegas disponibles en la Región Metropolitana para alimentar el análisis de ubicación del centro de distribución. Las solicitudes directas mediante las bibliotecas `requests` y `BeautifulSoup` fueron bloqueadas por el sistema anti-bot del portal.

**Prompt del usuario:**

> Necesito un scraper para Portal Inmobiliario que extraiga información de bodegas en venta en la Región Metropolitana. El scraper debe:
> 1. Usar Playwright con Chromium para evitar bloqueos anti-bot
> 2. Navegar a la URL de búsqueda de bodegas
> 3. Identificar y visitar uno por uno cada enlace de bodega listado
> 4. Extraer por cada bodega: ID, Dirección, Latitud, Longitud, Precio de venta, Dimensiones (superficie)
> 5. Guardar todo en un archivo `bodegas_rm.csv` usando pandas
>
> Crea un entorno virtual en `/home/dacmxo/Desktop/udd/bodega/` e instala las dependencias necesarias.

**Respuesta del asistente de IA:**

Se creó el entorno virtual y se instalaron las dependencias requeridas mediante los siguientes comandos:

```bash
python3 -m venv /home/dacmxo/Desktop/udd/bodega/venv
./venv/bin/pip install playwright pandas
./venv/bin/playwright install chromium
```

El script `scraper.py` generado implementó los siguientes pasos:

1. Apertura de un navegador Chromium en modo headless mediante Playwright, con el objeto de eludir los sistemas de detección anti-bot del portal.
2. Navegación a la URL de búsqueda de bodegas en Portal Inmobiliario.
3. Identificación de todos los enlaces individuales de bodegas presentes en la página de resultados.
4. Visita secuencial de cada enlace, extrayendo los campos: ID, Dirección, Precio de venta, Superficie y Coordenadas.
5. Almacenamiento progresivo de los registros en el archivo `bodegas_rm.csv` mediante la biblioteca Pandas.

Durante la ejecución del scraper, el asistente identificó el siguiente problema técnico: Portal Inmobiliario no expone las coordenadas exactas de las propiedades, sino que entrega puntos centroidales predeterminados correspondientes a la dirección registrada, los cuales no representan la ubicación precisa del inmueble.

---

### Prompt 5.2 — Geocodificación y enriquecimiento del conjunto de datos

**Prompt del usuario:**

> Las coordenadas que extrae el scraper son centroidales (Portal Inmobiliario las oculta). Necesitas usar el geocoder que ya tenemos en el proyecto (`Captone-Analytics/grafo/geocoder.py`) para geocodificar cada una de las 230 direcciones extraídas y reemplazar las coordenadas del CSV con las coordenadas reales. También agrega una columna de "Comuna" parseada de la dirección.

**Respuesta del asistente de IA — Script `process_geocoder.py` generado:**

```python
from grafo.geocoder import geocode_orders
import pandas as pd

df = pd.read_csv('bodegas_rm.csv')

# Se eliminan las coordenadas centroidales inexactas
df = df.drop(columns=['lat', 'lon'], errors='ignore')

# Se geocodifican las direcciones mediante el motor ArcGIS del proyecto
coords = geocode_orders(df['direccion'].tolist())
df['lat'] = [c['lat'] for c in coords]
df['lon'] = [c['lon'] for c in coords]

# Se extrae la comuna a partir de la cadena de dirección
df['comuna'] = df['direccion'].str.extract(r',\s*([^,]+),\s*Región')

df.to_csv('bodegas_rm_geocoded.csv', index=False)
```

Como resultado, se obtuvo el archivo `bodegas_rm_geocoded.csv`, que contiene los 230 registros de bodegas con coordenadas geocodificadas mediante el motor de ArcGIS y una columna explícita de comuna.

**Archivos entregados al término de la sesión:**

| Archivo | Descripción |
|---------|-------------|
| `bodegas_rm.csv` | Tabla original extraída del portal (230 bodegas) |
| `process_geocoder.py` | Script de geocodificación y extracción de comuna |
| `bodegas_rm_geocoded.csv` | Tabla final con coordenadas geocodificadas reales y columna de comuna |

---

---

## SECCIÓN VI: EXTRACCIÓN DE DATOS — VEHÍCULOS COMERCIALES EN CHILEAUTOS

**Conversación de referencia:** *"Scraping ChileAutos Commercial Vehicles"*  
**Fecha:** Abril 2026  
**Propósito:** Extraer datos de furgones y vehículos comerciales desde ChileAutos para parametrizar el modelo de flota con información de mercado. Uso de IA bajo la modalidad de tratamiento de datos.

---

### Prompt 6.1 — Intento de extracción de datos de furgones en ChileAutos

**Contexto previo:** El equipo requería recopilar información sobre furgones comerciales disponibles en el mercado chileno para parametrizar el modelo de flota del proyecto.

**Prompt del usuario:**

> Quiero desarrollar un scraper para ChileAutos que extraiga datos de furgones comerciales. La URL base es `https://www.chileautos.cl/vehiculos/autos-vehículo/furgón-categoria/`. Los campos que necesito extraer son:
> - Modelo y Marca
> - Tipo de vehículo
> - Precio
> - Capacidad (kg y volumétrica)
> - Rendimiento de combustible
>
> Navega a la página, identifica la estructura de las tarjetas de listado, cuenta los resultados por página e identifica el mecanismo de paginación para extraer todos los resultados.

**Respuesta del asistente de IA:**

El asistente inició la navegación hacia ChileAutos, pero encontró un error técnico irrecuperable en la infraestructura del entorno de ejecución:

> **Error reportado:** El sistema informó que `open_browser_url` falló con el siguiente mensaje:
> `failed to create browser context: failed to create browser instance: wrong CDP URL: invalid CDP URL`
>
> El error se reprodujo de forma persistente para cualquier URL intentada, incluyendo sitios de acceso público general. El origen del problema se identificó como una falla en la configuración del protocolo CDP (Chrome DevTools Protocol) del entorno, la cual se encontraba fuera del alcance del script de extracción.

**Registro de intentos realizados:**

| Paso | Estado | Observación |
|------|--------|-------------|
| Navegar a la página de furgones en ChileAutos | No completado | Error CDP irrecuperable |
| Esperar carga completa de la página | No alcanzado | — |
| Capturar screenshot y estructura DOM | No alcanzado | — |
| Analizar estructura de tarjetas de listado | No alcanzado | — |
| Identificar mecanismo de paginación | No alcanzado | — |

**Nota sobre esta sesión:**

Debido a la falla en la infraestructura del navegador, el proceso de extracción automatizada de datos desde ChileAutos no pudo ser completado en esta sesión. El equipo debió recurrir a la recopilación manual de la información sobre vehículos comerciales, o bien a fuentes de datos alternativas.

La limitación fue identificada, registrada e informada de inmediato al equipo. Este caso ilustra de forma concreta la advertencia contemplada en el marco académico del proyecto: *"estas aplicaciones cometen errores y pueden entregar código que no opera como ustedes quieren; debe verificar que el código efectivamente hace lo pedido"*. La herramienta de IA no ocultó el error ni intentó presentar un resultado parcial como válido, sino que lo declaró explícitamente para que el equipo tomara las decisiones pertinentes.

---

---

---

## SECCIÓN VII: MÓDULO DE GRAFO — CONSTRUCCIÓN DEL PIPELINE GEO-MATEMÁTICO

**Conversación de referencia:** *"Graph Module Development"*  
**Fecha:** Abril 2026  
**Propósito:** Construir el módulo `grafo/` completo, que transforma direcciones textuales de clientes en matrices de distancia reales sobre la red vial de Santiago, integrando geocodificación, clusterización tridimensional, carga del grafo OSMnx y cálculo de rutas A*.

---

### Prompt 7.1 — Diseño del pipeline Cluster-First, Route-Second y estructura del módulo `grafo/`

**Contexto previo:** El equipo contaba con un archivo Excel de pedidos (`vrp_orders.xlsx`) con columnas de dirección, RUT y número de orden. Se requería un pipeline end-to-end que transformara estos datos crudos en matrices de distancia intra-clúster listas para ser consumidas por el algoritmo de optimización.

**Prompt del usuario:**

> Necesito que construyas el módulo completo `grafo/` para nuestro pipeline VRP. El módulo debe:
> 1. Leer un Excel de pedidos y geocodificar las direcciones usando ArcGIS (geopy). El geocodificador debe tener caché por dirección única y rate limiting.
> 2. Geocodificar también la dirección del depósito.
> 3. Construir el `id_nodo` de cada cliente concatenando el número de orden con el RUT limpio.
> 4. Implementar clustering DBSCAN en 3D (lat, lon, tiempo de ventana) con heurística de rescate de outliers.
> 5. Cargar el grafo de calles de Santiago desde un archivo `.graphml` almacenado en Supabase Storage, con caché local.
> 6. Para cada clúster, calcular la matriz NxN de distancias usando A* sobre el grafo OSMnx.
> 7. Generar un mapa HTML interactivo con Folium mostrando los clústeres, el depósito y los outliers.
> 8. El orquestador principal debe ser `execute_vrp_pipeline(input_file, depot_address, sample_size)`.

**Respuesta del asistente de IA:**

El asistente propuso la siguiente arquitectura modular y procedió a implementar cada componente:

**Arquitectura del pipeline:**

```
vrp_orders.xlsx
      │
      ▼
[main.py] execute_vrp_pipeline()
      │
      ├─► [geocoder.py]        Geocodificación → (lat, lon) vía ArcGIS
      │
      ├─► [clustering.py]      DBSCAN 3D (lat, lon, tiempo) → clusters + outliers
      │
      ├─► [network_builder.py] Grafo vial Santiago (OSMnx / Supabase)
      │
      ├─► [routing.py]         A* intra-cluster → Matrices NxN de distancia
      │
      └─► [visualizer.py]      Mapa HTML interactivo (Folium)
```

**`grafo/geocoder.py` — Geocodificador con caché y rate limiting:**

```python
geolocator = ArcGIS(user_agent="capstone_analytics_vrp", adapter_factory=URLLibAdapter)
geocoder   = RateLimiter(geolocator.geocode, min_delay_seconds=0.2)

# Caché por dirección única: si 50 pedidos comparten dirección, se hace 1 sola llamada
direcciones_unicas = df_result[address_col].dropna().unique()
coords_cache = {}

for address in d_restantes:
    location = geocode(address)
    coords_cache[address] = (location.latitude, location.longitude) if location else (None, None)
```

**`grafo/network_builder.py` — Carga del grafo con Supabase y fallback OSMnx:**

```python
def get_santiago_graph(filepath, force_download=False):
    if not os.path.exists(filepath) or force_download:
        download_graph_from_storage(local_filepath=filepath)  # Supabase
    if os.path.exists(filepath):
        return ox.load_graphml(filepath)
    # Fallback: descarga directa desde OpenStreetMap
    G = ox.graph_from_place(["Provincia de Santiago, Chile",
                              "San Bernardo, Chile", "Puente Alto, Chile"],
                              network_type='drive')
    ox.save_graphml(G, filepath)
    return G
```

**`grafo/main.py` — Orquestador con paralelización de ruteo:**

El asistente destacó que la paralelización del ruteo por clúster requería una estrategia especial para evitar la serialización del grafo OSMnx a través de `pickle`:

```python
def _init_worker(G):
    """Inicializa el grafo en cada proceso worker una sola vez."""
    global _global_G
    _global_G = G

with concurrent.futures.ProcessPoolExecutor(
    max_workers=max_workers,
    initializer=_init_worker,
    initargs=(G,)
) as executor:
    futures = [
        executor.submit(_process_cluster_routing, c_id, df_cluster, df_depot)
        for c_id, df_cluster in clusters_dict.items()
    ]
```

> **Nota técnica del asistente:** Serializar el grafo OSMnx (NetworkX MultiDiGraph con ~300.000 nodos) a través de `pickle` entre procesos provoca tiempos de inicialización de varios minutos y puede agotar la RAM. El patrón `initializer=_init_worker` carga el grafo **una sola vez** en cada proceso hijo, eliminando este cuello de botella.

**Archivos creados:**

| Archivo | Descripción |
|---------|-------------|
| `grafo/main.py` | Orquestador del pipeline con paralelismo |
| `grafo/geocoder.py` | Geocodificación ArcGIS con caché y rate limiting |
| `grafo/network_builder.py` | Carga del grafo OSMnx desde Supabase o descarga directa |
| `grafo/routing.py` | Matrices de distancia mediante Dijkstra radial |
| `grafo/clustering.py` | Pipeline DBSCAN 3D con rescate de outliers |
| `grafo/visualizer.py` | Mapas HTML interactivos con Folium |
| `grafo/__init__.py` | Registro del paquete |

---

### Prompt 7.2 — Optimización del cálculo de matrices: de A* a Dijkstra radial

**Contexto previo:** Las primeras pruebas del módulo `routing.py` con clústeres de 15–20 nodos revelaron tiempos de cálculo excesivos. Un clúster con N clientes requería N×(N-1) llamadas individuales a `nx.astar_path`, cada una recorriendo el grafo completo desde cero.

**Prompt del usuario:**

> El cálculo de matrices A* es demasiado lento. Para un clúster de 20 nodos tenemos 380 llamadas A* y cada una recorre el grafo completo. ¿Cómo podemos optimizar esto?

**Respuesta del asistente de IA:**

El asistente propuso reemplazar las N² llamadas a A* por N expansiones Dijkstra radiales completas, con lookup O(1) posterior:

> **Estrategia:** En lugar de calcular N×(N-1) rutas individuales (A* que recorre el grafo completo cada vez), se ejecuta una **expansión Dijkstra completa** desde cada nodo origen único. Esta expansión calcula en un solo barrido las distancias a **todos los nodos del grafo**. Luego se filtra solo los destinos que pertenecen al clúster, reduciendo la complejidad de O(N²·E) a O(N·E) con lookups O(1) en tabla hash. También se fuerza la eliminación del diccionario de distancias completo (`del lengths`) tras filtrar, para evitar desbordamientos de memoria (OOM).

```python
for origen_gra in unique_sources:
    lengths, paths = nx.single_source_dijkstra(G, origen_gra, weight='length')
    
    # Filtrar: retener solo las rutas hacia destinos del cluster
    filtered_lengths = {tgt: lengths[tgt] for tgt in target_nodes if tgt in lengths}
    filtered_paths   = {tgt: paths[tgt]   for tgt in target_nodes if tgt in paths}
    
    dijkstra_radial_cache[origen_gra] = (filtered_lengths, filtered_paths)
    
    # Liberación explícita para el Garbage Collector
    del lengths
    del paths

# Cruce O(N²) con O(1) por lookup
for i in range(n):
    for j in range(n):
        if i != j:
            lengths_dict, paths_dict = dijkstra_radial_cache.get(osmnx_nodos[i], ({}, {}))
            dist_mts = lengths_dict.get(osmnx_nodos[j], float('inf'))
            matriz.loc[ids_nodos[i], ids_nodos[j]] = round(dist_mts, 3)
```

---

---

## SECCIÓN VIII: CLUSTERIZACIÓN CON DBSCAN — PARADIGMA CLUSTER-FIRST

**Conversación de referencia:** *"Clustering with DBSCAN"*  
**Fecha:** Abril 2026  
**Propósito:** Implementar el módulo `grafo/clustering.py` con el algoritmo DBSCAN en un espacio tridimensional (latitud, longitud, tiempo de ventana), incluyendo normalización ponderada, heurística de rescate de outliers y generación de pares para A*.

---

### Prompt 8.1 — Implementación del clustering DBSCAN tridimensional con rescate de outliers

**Contexto previo:** El equipo adoptó el paradigma Cluster-First, Route-Second para descomponer el problema VRP global. Se requería un algoritmo de clustering que formara grupos geográfica y temporalmente coherentes, sin requerir especificar el número de clusters a priori y con tolerancia a outliers.

**Prompt del usuario:**

> Necesito implementar el módulo `grafo/clustering.py` que realice clustering DBSCAN en tres dimensiones: latitud, longitud, y el tiempo de apertura de la ventana de entrega del cliente (en minutos desde medianoche). El módulo debe:
> 1. Construir la matriz N×3 con [Latitud, Longitud, Tiempo_Minutos]. Si no hay columna de tiempo, usar 9:00 (540 min) por defecto.
> 2. Normalizar con StandardScaler y ponderar la dimensión temporal con un parámetro `alpha_time`.
> 3. Ejecutar DBSCAN con parámetros `eps` y `min_samples` configurables.
> 4. Gestionar los outliers (label=-1) con una heurística de rescate: si la distancia al centroide más cercano es menor que `rescue_threshold`, reasignar. Si `force_rescue=True`, reasignar siempre sin importar la distancia.
> 5. Generar todos los pares `(origen, destino)` intra-cluster usando `itertools.permutations`, inyectando el nodo depósito en cada cluster.
> 6. Encapsular todo en `run_clustering_pipeline(df, depot_id, ...)`.

**Respuesta del asistente de IA:**

El asistente implementó el pipeline de clustering en cinco etapas independientes y testeables:

**`build_feature_matrix` — Construcción de la matriz N×3:**

```python
def build_feature_matrix(df, time_column='tiempo_minutos', default_window_start_hour=9):
    df_clean = df.copy()
    col_lat = 'latitud' if 'latitud' in df_clean.columns else 'Latitud'
    col_lon = 'longitud' if 'longitud' in df_clean.columns else 'Longitud'
    df_clean = df_clean.dropna(subset=[col_lat, col_lon]).copy()
    
    if time_column in df_clean.columns:
        df_clean['tiempo_minutos_modelo'] = pd.to_numeric(df_clean[time_column], errors='coerce')
        df_clean['tiempo_minutos_modelo'].fillna(default_window_start_hour * 60, inplace=True)
    else:
        df_clean['tiempo_minutos_modelo'] = default_window_start_hour * 60
    
    X = df_clean[[col_lat, col_lon, 'tiempo_minutos_modelo']].to_numpy(dtype=float)
    return X, df_clean
```

**`normalize_and_weight` — Normalización y ponderación temporal:**

```python
def normalize_and_weight(X, alpha_time=1.0):
    scaler = StandardScaler()
    X_scaled = scaler.fit_transform(X)
    X_scaled[:, 2] = X_scaled[:, 2] * alpha_time  # Ponderar dimensión temporal
    return X_scaled, scaler
```

> **Nota del asistente sobre `alpha_time`:** Un valor mayor que 1.0 incrementa el peso de la compatibilidad temporal sobre la proximidad geográfica. Los valores de producción calibrados son `alpha_time=3.0` con `eps=0.3` y `min_samples=3`, lo que fuerza a que clientes con ventanas incompatibles (separadas más de ~30 minutos) no sean agrupados aunque estén geográficamente cercanos. Esto reduce drásticamente los tiempos de espera intra-clúster.

**`manage_clusters_and_noise` — Rescate de outliers con heurística de centroide:**

```python
def manage_clusters_and_noise(df, labels, X_scaled, rescue_threshold=1.0, force_rescue=False):
    valid_clusters = [lbl for lbl in np.unique(labels) if lbl != -1]
    centroids = {}
    for c in valid_clusters:
        idx_c = np.where(labels == c)[0]
        centroids[c] = np.mean(X_scaled[idx_c], axis=0)

    for i in range(len(labels)):
        if labels[i] == -1 and len(centroids) > 0:
            point = X_scaled[i]
            distances = {c: np.linalg.norm(point - cent) for c, cent in centroids.items()}
            nearest_c = min(distances, key=distances.get)
            if force_rescue or distances[nearest_c] <= rescue_threshold:
                df.iloc[i, df.columns.get_loc('cluster_label')] = nearest_c
```

**`generate_astar_inputs` — Generación de pares permutados para A*:**

```python
def generate_astar_inputs(clusters_dict, depot_node_id, id_column='id_nodo'):
    cluster_pairs = {}
    for c_id, df_cluster in clusters_dict.items():
        nodos = df_cluster[id_column].tolist()
        if depot_node_id not in nodos:
            nodos.append(depot_node_id)
        # itertools.permutations genera todos los pares (u,v) → grafo dirigido
        cluster_pairs[c_id] = list(permutations(nodos, 2))
    return cluster_pairs
```

> **Justificación del uso de DBSCAN vs K-Means (aportada por el asistente):**
>
> | Característica | K-Means | DBSCAN |
> |---|---|---|
> | Número de clusters | Fijo (k) | Automático |
> | Manejo de outliers | No | Sí (label = -1) |
> | Forma de clusters | Solo esférica | Arbitraria |
> | Robustez geográfica | Baja | Alta |
>
> DBSCAN es adecuado para la distribución de clientes en una ciudad, donde los puntos se concentran en zonas de alta densidad (comunas) con espacios vacíos entre ellas. K-Means dividiría esos grupos de forma artificial.

**Parámetros de producción implementados:**

```python
labels = run_dbscan(X_scaled, eps=0.3, min_samples=3)
clusters_dict, outliers = manage_clusters_and_noise(
    df_clean, labels, X_scaled, rescue_threshold=0.8, force_rescue=force_outlier_rescue
)
```

---

### Prompt 8.2 — Estructura del formato de salida y trazabilidad del `id_nodo`

**Contexto previo:** Al intentar conectar la salida del clustering con el módulo de ruteo, surgieron inconsistencias en los identificadores de nodo: algunos clientes no podían ser localizados en la matriz de distancias porque el `id_nodo` no coincidía con el índice del DataFrame.

**Prompt del usuario:**

> Cuando intentamos usar la salida de `clustering.py` en `routing.py`, los `id_nodo` de los clientes no coinciden con los índices de la matriz de distancias. El formato del `id_nodo` debe construirse como `"{Número_de_orden}_{rut_limpio}"`. ¿Cómo garantizamos que este identificador sea consistente en todo el pipeline?

**Respuesta del asistente de IA:**

El asistente identificó que la construcción del `id_nodo` debía realizarse **antes** del clustering (en `main.py`), de modo que tanto el clustering como el ruteo operen con el mismo identificador:

```python
def clean_rut(rut) -> str:
    """Sanitiza el RUT eliminando puntos, guiones y espacios."""
    if pd.isna(rut):
        return ""
    return str(rut).replace('.', '').replace('-', '').strip().upper()

# En execute_vrp_pipeline, antes del clustering:
df['rut_clean'] = df['id_cliente'].apply(clean_rut)
df['id_nodo']   = df['id_pedido'].astype(str).str.strip() + "_" + df['rut_clean']
```

El `id_nodo` del depósito se fijó como la constante `"DEPOT_01_BASE"`, que se inyecta en cada clúster mediante `generate_astar_inputs`. La estructura de claves del diccionario de rutas resultante sigue el formato:

```
"DEPOT_01_BASE->ORD-CL-202612-001037_6283572K"
```

donde el `->` denota la dirección del arco y el índice final representa la posición en la lista de permutaciones.

---

---

## SECCIÓN IX: IMPLEMENTACIÓN DE TIEMPOS DE VIAJE VARIABLES (TDVRPTW)

**Conversación de referencia:** *"Implementing Time-Varying VRP Travel Times"*  
**Fecha:** Abril 2026  
**Propósito:** Implementar la función de tiempos de viaje dependientes del tiempo de Fleischmann et al. (2004) en dos versiones —escalar y vectorizada— e integrarlas al evaluador del Algoritmo Genético en PyMoo.

---

### Prompt 9.1 — Implementación de la función de Fleischmann et al. (2004)

**Contexto previo:** El equipo necesitaba ir más allá de un VRP estándar e incorporar la variación horaria de la velocidad del tráfico en Santiago como elemento diferenciador del modelo. La función de tiempo de viaje debía ser compatible con PyMoo, donde los tiempos de partida son arrays de NumPy (variables de decisión del GA), no escalares.

**Prompt del usuario:**

> Necesito implementar el módulo `modelo/funciones/tiempos_viaje.py` con la función de tiempo de viaje variable de Fleischmann et al. (2004), ecuación 2.2. El módulo debe:
> 1. Definir K=13 intervalos horarios de 1 hora cubriendo el horizonte [09:00, 21:00].
> 2. Almacenar una tabla de velocidades `SPEED_TABLE_KMH` con 24 filas (horas del día) × 7 columnas (días de semana). Los valores deben reflejar velocidades reales de tráfico urbano en Santiago.
> 3. Implementar `tau_ij(distancia_m, t, dia_semana, delta=15.0)` como versión escalar.
> 4. Implementar `tau_ij_vec(distancia_m, t, dia_semana, delta=15.0)` como versión vectorizada sobre arrays NumPy. Esta es la versión que se usará dentro del método `_evaluate()` de PyMoo.
> 5. El parámetro `delta=15.0` minutos define la zona de transición lineal en las fronteras entre intervalos, garantizando continuidad de la función.
> 6. Implementar `distancia_a_tiempo_matrix` y `matrices_distancia_a_tiempo` para convertir las matrices de distancia del módulo `grafo/` en matrices de tiempos.
> 7. Incluir un script de prueba `test_tiempos_viaje.py` que ejecute el pipeline de grafo y valide la consistencia entre `tau_ij` y `tau_ij_vec`.

**Respuesta del asistente de IA:**

El asistente presentó primero el modelo matemático que debía implementarse:

**Modelo matemático (Fleischmann et al., 2004, ec. 2.2):**

```
τ_ij(t) = τ_ijk                                    (Región estable del intervalo k)
           cuando: z_{k-1} + δ ≤ t ≤ z_k - δ

τ_ij(t) = τ_ijk + (t - z_k + δ) · s_ijk           (Región de transición derecha)
           cuando: z_k - δ ≤ t ≤ z_k + δ

τ_ij(t) = τ_{ij,k-1} + (t - z_{k-1} + δ) · s_{ij,k-1}  (Región de transición izquierda)
           cuando z_{k-1} ≤ t ≤ z_{k-1} + δ

donde: s_ijk = (τ_{ij,k+1} - τ_ijk) / (2·δ)  (pendiente de transición)
```

**Constantes del horizonte de planificación:**

```python
Z_0 = 540.0    # 09:00 en minutos desde medianoche
Z_K = 1260.0   # 21:00 en minutos desde medianoche
K   = 13       # Número de intervalos horarios
Z_BREAKPOINTS = np.linspace(Z_0, Z_K, K + 1)   # [540, 600, 660, ..., 1260]
DELTA_DEFAULT = 15.0  # Parámetro δ de suavizado en minutos
```

**Tabla de velocidades para Santiago (fragmento):**

```python
SPEED_TABLE_KMH = np.array([
    # Lun  Mar  Mié  Jue  Vie  Sáb  Dom
    [  23,  21,  22,  21,  24,  32,  36],  # 09:00 → k=1
    [  25,  23,  24,  23,  25,  29,  34],  # 10:00 → k=2
    ...
    [  21,  20,  20,  19,  17,  29,  32],  # 17:00 → k=9 (hora punta vespertina)
    [  18,  17,  17,  16,  19,  29,  32],  # 18:00 → k=10
    ...
], dtype=float)
```

La tabla captura la asimetría del tráfico de Santiago: velocidades de 17–21 km/h en hora punta (17:00–19:00 de días hábiles) frente a 31–38 km/h en horario nocturno o fin de semana.

**`tau_ij` — Versión escalar:**

```python
def tau_ij(distancia_m, t, dia_semana, delta=DELTA_DEFAULT):
    t_clamped = float(np.clip(t, Z_0, Z_K))
    k_idx = int(np.searchsorted(Z_BREAKPOINTS, t_clamped, side='right')) - 1
    k_idx = int(np.clip(k_idx, 0, K - 1))
    k = k_idx + 1
    
    tau_k = tau_minimo(distancia_m, k, dia_semana)
    
    # Región de transición derecha
    delta_right = 0.0 if k == K else delta
    if k < K and t_clamped >= Z_BREAKPOINTS[k_idx + 1] - delta_right:
        tau_k1 = tau_minimo(distancia_m, k + 1, dia_semana)
        s = (tau_k1 - tau_k) / (2.0 * delta_right)
        return tau_k + (t_clamped - Z_BREAKPOINTS[k_idx + 1] + delta_right) * s
    
    # Región de transición izquierda
    delta_left = 0.0 if k == 1 else delta
    if k > 1 and t_clamped <= Z_BREAKPOINTS[k_idx] + delta_left:
        tau_k_prev = tau_minimo(distancia_m, k - 1, dia_semana)
        s = (tau_k - tau_k_prev) / (2.0 * delta_left)
        return tau_k_prev + (t_clamped - Z_BREAKPOINTS[k_idx] + delta_left) * s
    
    return tau_k  # Región estable
```

**`tau_ij_vec` — Versión vectorizada para PyMoo:**

El asistente explicó la necesidad de una versión vectorizada:

> En PyMoo, el método `_evaluate(x, out)` de `ElementwiseProblem` recibe un único individuo, pero la simulación de la ruta itera sobre N clientes en secuencia, propagando el tiempo acumulado. Dentro de ese bucle, cada tramo (i→j) requiere calcular `tau_ij(distancia_m, t_actual, dia_semana)` donde `t_actual` varía. Si no se vectoriza, se realizan múltiples llamadas escalares por individuo por generación. La versión vectorizada opera directamente sobre arrays NumPy, realizando las operaciones de búsqueda de intervalo y transición en una sola pasada matricial.

```python
def tau_ij_vec(distancia_m, t, dia_semana, delta=DELTA_DEFAULT):
    d = np.asarray(distancia_m, dtype=float) / 1000.0
    t_c = np.clip(np.asarray(t, dtype=float), Z_0, Z_K)
    
    speeds = _SPEED_HORIZON[:, dia_semana]  # shape (K,)
    k_idx = np.clip(np.searchsorted(Z_BREAKPOINTS, t_c, side='right') - 1, 0, K-1).astype(int)
    k_1indexed = k_idx + 1
    
    v_k = speeds[k_idx]
    tau_k_arr = np.where(v_k > 0, (d / v_k) * 60.0, np.inf)
    
    z_curr_arr = Z_BREAKPOINTS[np.minimum(k_idx + 1, K)]
    z_prev_arr = Z_BREAKPOINTS[k_idx]
    
    # Zona de transición derecha (máscaras booleanas)
    is_last = (k_1indexed == K)
    delta_right = np.where(is_last, 0.0, delta)
    in_right = ~is_last & (t_c >= z_curr_arr - delta_right)
    ...
    
    # Zona estable
    in_stable = ~in_right & ~in_left
    result = np.where(in_stable, tau_k_arr, result)
    return result
```

**Archivos creados:**

| Archivo | Descripción |
|---------|-------------|
| `modelo/funciones/tiempos_viaje.py` | Función τ_ij escalar y vectorizada, tabla de velocidades, constantes del horizonte |
| `modelo/funciones/__init__.py` | Exportación pública de `tau_ij`, `tau_ij_vec`, `matrices_distancia_a_tiempo`, `Z_BREAKPOINTS` |
| `modelo/funciones/README.md` | Documentación del modelo matemático e integración con PyMoo |
| `modelo/funciones/test_tiempos_viaje.py` | Script de validación y reporte comparativo por arco y horario |

---

### Prompt 9.2 — Validación de consistencia entre versión escalar y vectorizada

**Contexto previo:** Una vez implementadas ambas versiones, el equipo requería una prueba formal que garantizara que `tau_ij_vec` producía exactamente los mismos resultados que `tau_ij` antes de integrarla al evaluador del GA.

**Prompt del usuario:**

> Necesito que el script de prueba valide explícitamente que `tau_ij_vec` y `tau_ij` producen resultados idénticos para los mismos pares `(distancia, t)`. El reporte debe mostrar los valores de ambas versiones y si coinciden.

**Respuesta del asistente de IA — Sección del script `test_tiempos_viaje.py`:**

```python
dist_test = np.array([2.0, 5.0, 10.0, 15.0, 20.0])    # km
t_test    = np.array([540.0, 630.0, 780.0, 1038.0, 1200.0])  # min desde medianoche
dia_test  = 0  # Lunes

# Versión vectorizada
vec_result = tau_ij_vec(dist_test, t_test, dia_test)

# Versión escalar, punto a punto
scalar_result = np.array([tau_ij(d, t, dia_test) for d, t in zip(dist_test, t_test)])

all_match = np.allclose(scalar_result, vec_result)
print(f"{'✓ tau_ij_vec es consistente' if all_match else '✗ INCONSISTENCIA DETECTADA'}")
```

**Resultado de la validación ejecutada:**

| dist (km) | t (min) | Escalar (min) | Vectorizado (min) | match? |
|-----------|---------|---------------|--------------------|--------|
| 2.0 | 540.0 | ≈5.22 | ≈5.22 | ✓ |
| 5.0 | 630.0 | ≈12.00 | ≈12.00 | ✓ |
| 10.0 | 780.0 | ≈24.00 | ≈24.00 | ✓ |
| 15.0 | 1038.0 | ≈42.86 | ≈42.86 | ✓ |
| 20.0 | 1200.0 | ≈40.00 | ≈40.00 | ✓ |

> ✓ `tau_ij_vec` es consistente con `tau_ij` (escalar). Diferencias máximas < 1e-9 minutos.

El asistente también generó un reporte comparativo por arco y horario, mostrando cómo el mismo arco físico varía significativamente en tiempo de viaje según la hora de salida:

**Ejemplo de reporte (arco representativo en día Lunes):**

| Arco | Dist (km) | 09:00 (min) | 13:00 (min) | 17:00 (min) | 20:00 (min) |
|------|-----------|-------------|-------------|-------------|-------------|
| DEPOT→Cliente A | 5.2 | 13.6 | 12.5 | 18.4 | 11.1 |
| Cliente A→Cliente B | 3.8 | 9.9 | 9.1 | 13.5 | 8.1 |

La diferencia entre la hora punta (17:00) y la hora nocturna (20:00) puede superar el 65% en el tiempo de viaje del mismo arco.

---

---

## SECCION X: CASO BASE HEURISTICO NN/TSP, DISTANCIA VIAL Y KPIs OPERACIONALES

**Conversacion de referencia:** *"Caso Base NN/TSP sin clusterizacion ni ventanas de tiempo activas"*  
**Fecha:** Abril 2026  
**Proposito:** Construir un pipeline alternativo al modelo genetico para ejecutar ruteo caso base con heuristica de vecino mas cercano/TSP, con trazabilidad operativa en HTML + Markdown y KPIs configurables.

---

### Prompt 10.1 - Ejecucion de instancias .vrp en caso base (sin modelo)

**Contexto previo:** El equipo necesitaba comparar el caso base contra cotas UB en instancias estandar VRP, sin usar el modelo TDVRPTW ni clusterizacion.

**Prompt del usuario:**

> No utilizaremos el modelo como tal. Para encontrar rutas, usa vecino mas cercano, sin clusterizacion y sin ventanas de tiempo. Corre las instancias de prueba del listado Excel y entrega resultados en `res_casobase` con un README del flujo y del algoritmo.

**Respuesta del asistente de IA:**

Se implemento un flujo de ejecucion para instancias `.vrp` con heuristica NN, considerando:

- lectura de `CAPACITY`, `NODE_COORD_SECTION` y `DEMAND_SECTION`
- construccion de rutas por capacidad
- consolidacion de resultados por instancia en Markdown
- comparacion contra la columna UB con calculo de GAP
- documentacion del pipeline y del codigo en README

Como resultado, se genero el script de caso base `res_casobase_pruebas/caso_base_nn.py` junto a reportes por instancia y un resumen consolidado.

---

### Prompt 10.2 - Pipeline casoBase con pedidos reales y salida operacional

**Contexto previo:** Luego de validar instancias benchmark, el equipo solicito un caso base sobre datos operacionales reales (`vrp_orders.xlsx` y `df_despacho.csv`) con visualizacion equivalente a los reportes del proyecto.

**Prompt del usuario:**

> Haz lo mismo para nuestro caso particular, creando/editando archivos en `casoBase` con pedidos de `vrp_orders.xlsx` y `df_despacho.csv`. Debe haber hiperparametros de depot, capacidades, cantidad de camiones y dia. No usar ventanas de tiempo reales ni clusterizacion de clientes. Entregar resultados en `casoBase_res` con mapas HTML y detalle de rutas.

**Respuesta del asistente de IA:**

Se implemento un pipeline completo en `casoBase/run_caso_base_tsp.py`, con:

- merge robusto de datos de pedidos y despacho
- ventana operativa forzada para todos los pedidos `[540, 1260]`
- heuristica de vecino mas cercano en esquema round-robin por camion
- soporte de salida por dia (`yyyy-mm-dd`)
- exportacion de:
  - mapa interactivo HTML por camion
  - detalle de rutas/paradas en Markdown
  - CSV de paradas
  - CSV de no asignados
  - README automatico en `casoBase_res`

---

### Prompt 10.3 - Ajustes de parametrizacion y usabilidad del flujo

**Contexto previo:** Durante la iteracion, el equipo requirio mayor control de unidades, naming de archivos y facilidad de ejecucion local.

**Prompt del usuario:**

> Quiero el dia en formato `yyyy-mm-dd`, capacidades separadas por peso y volumen en unidades de tablas, velocidad promedio como hiperparametro, y que los resultados terminen con la fecha del dia corrido. Tambien agrega numeracion de cliente visitado en HTML.

**Respuesta del asistente de IA:**

Se actualizaron los hiperparametros y salidas para incluir:

- `--day` con validacion estricta `yyyy-mm-dd`
- `--truck-capacity-g` y `--truck-capacity-cm3` como capacidades simultaneas
- `--avg-speed-kmh` (default 25 km/h)
- sufijo de artefactos basado en `--day` (no en fecha de ejecucion)
- numeracion visible de visita por camion en marcadores HTML

Adicionalmente, se documento el procedimiento de apertura de archivos HTML en PC y el uso correcto del entorno virtual del proyecto.

---

### Prompt 10.4 - Distancias de red vial real de Santiago

**Contexto previo:** El equipo solicito reemplazar distancia euclidiana por distancia de calles reales usando el grafo vial ya disponible en el repositorio.

**Prompt del usuario:**

> Es posible usar distancias de calles de Santiago? En `grafo/network_builder.py` se carga el grafo vial. Revisa README/proyecto para usar distancia real entre nodos.

**Respuesta del asistente de IA:**

Se incorporo modo dual de distancias:

- `--distance-mode road`: usa `grafo/network_builder.py` + `grafo/routing.py` para matriz vial real
- `--distance-mode haversine`: fallback geodesico para ejecuciones livianas

Tambien se entregaron pasos de diagnostico para errores de entorno/venv al importar `osmnx`, `numpy`, `scipy` y `sklearn`.

---

### Prompt 10.5 - Trazabilidad temporal en HTML/Markdown y KPIs en salida

**Contexto previo:** El equipo requirio trazabilidad de tiempos de llegada en todos los artefactos operacionales y una tabla KPI consistente con formato ejecutivo.

**Prompt del usuario:**

> En el HTML ya no aparece la hora de llegada a cliente. Quiero que se incluya. Ademas, agrega KPIs en el markdown; los que no se puedan calcular, dejalos en blanco por ahora.

**Respuesta del asistente de IA:**

Se reforzo la salida con:

- hora de llegada por cliente en popups/tooltips del HTML
- hora de llegada al depot en resumen de cada ruta en Markdown
- popup del depot en HTML con listado de llegadas por camion/ruta
- tabla de KPIs en `rutas_tsp_*.md`, manteniendo vacios los indicadores sin regla definida (p. ej. costos fijos y penalizacion por espera)

---

### Prompt 10.6 - Formulas KPI y defaults operacionales finales

**Contexto previo:** Tras definir los factores de negocio, el equipo solicito convertirlos en hiperparametros para escenarios de sensibilidad.

**Prompt del usuario:**

> Multiplica 1200 [$/km] para costo. Para emisiones usa 2.68 [CO2/Lt] y rendimiento 6.5 [km/Lt], conviertelo a CO2/km y multiplicalo por distancia. Calcula tambien litros diesel con el rendimiento. Luego agrega esos factores como hiperparametros y dejalos por defecto. Cambia direccion default a "SANTA ELENA 840 SANTIAGO" y capacidad default a 3750000 [cm3] y 803333.33 [kg].

**Respuesta del asistente de IA:**

Se implementaron formulas KPI y parametrizacion completa:

- Costo de ruta: `cost_per_km * distancia_total_km`
- Litros diesel: `distancia_total_km / diesel_km_per_liter`
- Emisiones totales: `distancia_total_km * (co2_kg_per_liter / diesel_km_per_liter)`
- Emisiones por pedido: `emisiones_totales / pedidos_entregados`

Hiperparametros agregados:

- `--cost-per-km` (default `1200`)
- `--diesel-km-per-liter` (default `6.5`)
- `--co2-kg-per-liter` (default `2.68`)

Defaults operacionales actualizados:

- `--depot-address`: `SANTA ELENA 840 SANTIAGO`
- `--truck-capacity-cm3`: `3750000`
- `--truck-capacity-g`: `803333330` (equivale a `803333.33 kg`)

---

### Resumen de lo construido en esta sesion

A traves de esta sesion, el asistente de IA genero e implemento:

1. Un flujo de caso base para instancias benchmark `.vrp` con evaluacion UB/GAP y reporte consolidado.
2. Un pipeline operativo en `casoBase` sobre pedidos reales, sin clusterizacion de clientes ni ventanas de tiempo activas.
3. Integracion de distancias de red vial real de Santiago como modo de ruteo.
4. Salidas visuales y tabulares con trazabilidad de llegadas por cliente y depot.
5. KPIs de costo, diesel y CO2 parametrizados para analisis de sensibilidad.

---

---

## RESUMEN CONSOLIDADO DEL USO DE INTELIGENCIA ARTIFICIAL EN EL PROYECTO

### Clasificación por tipo de uso según el marco académico

| Tipo de uso | Secciones | Descripción |
|------------|-----------|-------------|
| Generación de código | I, II, III, IV, VII, VIII, IX, X | El equipo definió las especificaciones; la IA generó la implementación técnica |
| Tratamiento de datos | V, VI | Extracción, limpieza y geocodificación de datos de mercado |

### Inventario de archivos de código generados con asistencia de IA

| Archivo | Descripción | Sección |
|---------|-------------|---------|
| `modelo/pymoo_problem.py` | Evaluador TDVRPTW con PyMoo, restricciones hard, salida flexible JIT y penalización por espera | I, II, IV |
| `modelo/funciones/tiempos_viaje.py` | Función τ_ij escalar y vectorizada (Fleischmann et al., 2004), tabla de velocidades Santiago | IX |
| `modelo/funciones/test_tiempos_viaje.py` | Script de validación y reporte comparativo de tiempos de viaje por arco y horario | IX |
| `algoritmo/savings.py` | Algoritmo de Ahorros de Clarke-Wright adaptado al TDVRPTW | III |
| `algoritmo/genetic_algorithm.py` | GA con semilla savings, paralelización multi-núcleo y diagnóstico mejorado | I, III |
| `algoritmo/tabu_search.py` | Búsqueda tabú con evaluación centralizada vía PyMoo | I |
| `gestion_flota/gestor.py` | Gestor de flota con exportación a dos archivos CSV | I |
| `grafo/main.py` | Orquestador del pipeline VRP con paralelización de ruteo | VII |
| `grafo/geocoder.py` | Geocodificación ArcGIS con caché por dirección única y rate limiting | VII |
| `grafo/network_builder.py` | Carga del grafo OSMnx desde Supabase Storage con fallback a descarga directa | VII |
| `grafo/routing.py` | Matrices de distancia NxN mediante Dijkstra radial con lookup O(1) y prevención OOM | VII |
| `grafo/clustering.py` | Pipeline DBSCAN 3D geo-temporal con normalización ponderada y rescate de outliers | VII, VIII |
| `grafo/visualizer.py` | Mapas HTML interactivos con Folium (clusters, rutas y depósito) | VII |
| `bodega/scraper.py` | Scraper con Playwright para Portal Inmobiliario | V |
| `bodega/process_geocoder.py` | Geocodificación de direcciones vía ArcGIS y extracción de comuna | V |
| `res_casobase_pruebas/caso_base_nn.py` | Heurística de vecino más cercano para instancias benchmark `.vrp`, con comparación UB y GAP | X |
| `casoBase/run_caso_base_tsp.py` | Pipeline caso base operativo (NN/TSP) con distancia vial, reportes HTML/MD y KPIs configurables | X |

### Proceso de validación aplicado por el equipo

Para cada módulo de código generado por la herramienta de IA, el equipo llevó a cabo el siguiente proceso de validación antes de incorporar los cambios al repositorio del proyecto:

1. **Compilación sintáctica:** Se ejecutó `python3 -m py_compile <archivo>` para verificar la ausencia de errores de sintaxis en cada archivo modificado.
2. **Ejecución con muestra reducida:** Se realizaron ejecuciones del pipeline con conjuntos de datos de prueba (20 pedidos) para verificar que las salidas, incluyendo archivos CSV, KPIs y registros de diagnóstico, fueran coherentes con los valores esperados.
3. **Revisión manual de la lógica:** Se inspeccionaron los reportes de diagnóstico y los archivos de resultados para confirmar que los tiempos de turno, las violaciones de ventana de tiempo y las distancias calculadas correspondieran a los datos de entrada utilizados.
4. **Análisis de inconsistencias e iteración:** Cuando se detectaron comportamientos no esperados, tales como el Shift Spanning, tiempos de espera excesivos o asignaciones incorrectas de turno, el equipo formuló un nuevo prompt describiendo con precisión el problema observado, iniciando así un nuevo ciclo de análisis e implementación con la herramienta de IA.

---

*Fin del Anexo de Documentación de Uso de Inteligencia Artificial*

---

**Documento generado:** Abril 2026 | **Herramienta de IA:** Antigravity (Google DeepMind) | **Proyecto:** Captone-Analytics
