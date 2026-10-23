# Documentación del Proyecto 2 — PacíficoExpress.gt
**Asignatura:** Modelación y Simulación 1  
**Semestre:** Segundo Semestre 2026  
**Nombre del Archivo:** `Documentacion_Proyecto2_G3.md`  

---

## Resumen Ejecutivo

El presente documento constituye la memoria técnica y analítica de la simulación del centro logístico **PacíficoExpress.gt**. El estudio evalúa el desempeño operativo, los niveles de servicio y la estructura de costos asociados al procesamiento, preparación, transporte interno y despacho de productos distribuidos en tres categorías principales (*Farmacéuticos*, *Textiles* y *Ferretería*) hacia tres nodos regionales (*Escuintla*, *Chimaltenango* y *Sacatepéquez*).

Mediante la herramienta de simulación **Simio 15**, se desarrollaron dos escenarios principales:
1. **Modelo Base (`[MyS1]Modelo1_G3.spfx`):** Representa las condiciones actuales de operación con horarios de trabajo divididos en jornadas semanales y de sábado, domingos cerrados, recursos compartidos limitados y políticas de enrutamiento estándar.
2. **Modelo Optimizado (`[MyS1]ModeloOptimizado_G3.spfx`):** Implementa estrategias de rebalanceo de capacidad, optimización de flota de transporte interno (AGVs) y eliminación del cuello de botella crítico identificado en la preparación de productos de ferretería, logrando una reducción sustancial en el costo operativo total diario y una mejora en los tiempos globales de permanencia en el sistema (*Flow Time*).

---

## 1. Descripción y Diseño del Sistema Actual (Modelo Base)

### 1.1 Flujo General del Sistema
El flujo operativo dentro de la bodega central de $30,000 \text{ m}^2$ sigue una arquitectura secuencial dividida en cuatro etapas principales:

1. **Recepción y Clasificación (Persona 1):** Entrada de productos e inspección inicial. Asignación del tipo de mercancía y clasificación por probabilidad según categoría.
2. **Preparación y Almacenamiento Temporal (Persona 2):** Procesamiento en las estaciones especializadas `PrepFarmaceuticos`, `PrepTextiles` y `PrepFerreteria`. Los productos aguardan en las colas de entrada (*Input Buffer* de capacidad 20 unidades) hasta contar con disponibilidad de estación y de uno de los 3 operarios compartidos (`OperariosAlmacen`).
3. **Transporte Interno mediante AGVs (Persona 3):** Transferencia automatizada de productos listos desde los nodos de salida `Output@` de cada estación de preparación hacia el nodo de descarga común `PuntoEntrega` mediante la flota de vehículos guiados automáticos (`AGV1` y `AGV2`) a través de la red `RedAGV`.
4. **Despacho Regional (Persona 6):** Transferencia por conector hacia `NodoDespacho` y enrutamiento hacia las tres estaciones regionales (`DespachoEscuintla`, `DespachoChimaltenango`, `DespachoSacatepequez`) operadas por personal dedicado, para su salida final en sus respectivos `Sink`.

```text
[Recepción / Clasificación]
            │
            ├───► PrepFarmaceuticos (Cola máx: 20) ──┐
            ├───► PrepTextiles      (Cola máx: 20) ──┼──► [RedAGV: AGV1/AGV2] ──► PuntoEntrega ──► NodoDespacho ──┬──► DespachoEscuintla ──► Salida
            └───► PrepFerreteria    (Cola máx: 20) ──┘                                                               ├──► DespachoChimaltenango ─► Salida
                                                                                                                     └──► DespachoSacatepequez ──► Salida
```

---

## 2. Supuestos, Procesos, Variables de Estado y Eventos

### 2.1 Supuestos Fundamentales del Modelo
* **Persistencia de Atributos:** La variable de estado `ModelEntity.Destino` se asigna una sola vez al ingresar la entidad al sistema y no se altera durante el transporte ni la preparación.
* **Política de Fin de Turno:** Las operaciones en ejecución al momento de finalizar la jornada laboral activa completan su procesamiento (`Off Shift Rule = Finish Work Already Started`). No se inician nuevas operaciones en períodos de descanso o fuera de turno.
* **Sin Costos Operativos los Domingos:** La capacidad programada de recursos y servidores los domingos es $0$, garantizando que las tarifas horarias e *Idle Cost Rates* no generen cargos financieros en el día de cierre.
* **Capacidad de Buffer Limitada:** Las colas de entrada en la fase de preparación poseen un límite de $20$ unidades para simular la capacidad física del almacenamiento temporal.

### 2.2 Variables de Estado y Atributos

| Variable / Atributo | Tipo | Objeto Asociado | Descripción / Valores |
|---|---|---|---|
| `ModelEntity.Destino` | Entero / Discreto | `ModelEntity` | Identificador del destino regional: $1 =$ Escuintla, $2 =$ Chimaltenango, $3 =$ Sacatepéquez. |
| `Capacity.ScheduledUtilization` | Real (porcentaje) | `Server` / `Resource` | Registro automático de utilización programada de estaciones y personal. |
| `InputBuffer.Contents.NumberInStation` | Entero | `Server` | Cantidad instantánea de productos en almacenamiento temporal. |
| `UnitsAllocated` | Entero | `Resource` | Cantidad de operarios asignados simultáneamente (máximo 3 en `OperariosAlmacen`). |

### 2.3 Principales Eventos Discretos del Sistema
1. **`Event_EntityArrival`:** Llegada de producto a recepción y determinación probabilística de categoría y destino regional.
2. **`Event_SeizeOperator`:** Asignación de 1 unidad de `OperariosAlmacen` para iniciar el procesamiento en la estación correspondiente.
3. **`Event_PreparationComplete`:** Liberación del operario de almacenamiento y transferencia del producto al `OutputBuffer` para solicitar un vehículo AGV.
4. **`Event_AGVDispatch`:** Asignación del vehículo más cercano (`AGV1` o `AGV2`) para transportar la entidad hacia `PuntoEntrega`.
5. **`Event_DispatchComplete`:** Salida definitiva del producto del centro logístico tras ser procesado en la estación regional correspondiente.

---

## 3. Configuración de Horarios, Recursos, Fallos y Finanzas

### 3.1 Horarios Operativos y Day Patterns
El calendario semanal (`HorarioOperarios`) aplica a las estaciones de preparación y al recurso de operarios compartidos:

* **`PatronSemana` (Lunes a Viernes):**
  * $06:00 - 12:00$ (Activo, Capacidad $= 3$)
  * $12:00 - 13:00$ (Almuerzo, Capacidad $= 0$)
  * $13:00 - 18:00$ (Activo, Capacidad $= 3$)
  * $18:00 - 06:00$ (Fuera de turno, Capacidad $= 0$)
* **`PatronSabado`:**
  * $08:00 - 12:00$ (Activo, Capacidad $= 3$)
  * $12:00 - 12:30$ (Almuerzo, Capacidad $= 0$)
  * $12:30 - 15:00$ (Activo, Capacidad $= 3$)
  * $15:00 - 08:00$ (Fuera de turno, Capacidad $= 0$)
* **Domingo:** Capacidad $= 0$ durante las 24 horas.

### 3.2 Parámetros de Preparación y Transporte

| Estación / Elemento | Distribución de Tiempo | Distancia a `PuntoEntrega` | Parámetros Financieros |
|---|---|---|---|
| `PrepFarmaceuticos` | $Random.Uniform(3.5, 5.5) \text{ min}$ | $60 \text{ m}$ (Ida / Vuelta) | Fijo: $\text{Q}200/\text{día}$, Variable: $\text{Q}3.00/\text{prod}$ |
| `PrepTextiles` | $Random.Uniform(3, 5) \text{ min}$ | $45 \text{ m}$ (Ida / Vuelta) | Fijo: $\text{Q}100/\text{día}$, Variable: $\text{Q}2.00/\text{prod}$ |
| `PrepFerreteria` | $Random.Uniform(5, 7) \text{ min}$ | $50 \text{ m}$ (Ida / Vuelta) | Fijo: $\text{Q}110/\text{día}$, Variable: $\text{Q}2.50/\text{prod}$ |
| `OperariosAlmacen` | 3 Unidades compartidas | N/A | Salario: $\text{Q}35.00/\text{hora}$ por operario |
| `AGV1` / `AGV2` | Velocidad: $1.0 \text{ m/s}$ | Rutas dinámicas en `RedAGV` | Fijo: $\text{Q}80/\text{día}$, Uso: $\text{Q}20/\text{hora}$ |

### 3.3 Regla de Despacho Regional
En `NodoDespacho`, la selección de ruta por conector hacia las estaciones regionales se determina mediante el peso lógico `Selection Weight`:
* **Escuintla ($45\%$ demanda):** `ModelEntity.Destino == 1`
* **Chimaltenango ($35\%$ demanda):** `ModelEntity.Destino == 2`
* **Sacatepéquez ($20\%$ demanda):** `ModelEntity.Destino == 3`

Cada servidor de despacho procesa en tiempo $Random.Uniform(2, 4) \text{ min}$ asistido por su respectivo operario dedicado.

---

## 4. Análisis de Resultados del Modelo Base (`[MyS1]Modelo1_G#.spfx`)

Tras ejecutar la simulación del modelo base durante un ciclo operativo de 24 horas y validar contra corridas extendidas, se obtuvieron las siguientes métricas del sistema:

### 4.1 KPIs de Operación Base

| Indicador clave (KPI) | Valor Registrado | Observación |
|---|---|---|
| **Productos totales ingresados a recepción** | $308 \text{ unidades}$ | Flujo de entrada registrado. |
| **Productos procesados y despachados** | $109 \text{ unidades}$ | Unidades con ciclo completo finalizado. |
| **Tiempo de simulación completado** | $24.00 \text{ horas}$ | Jornada evaluada. |
| **Costo Operativo Total Acumulado** | $\text{Q}2,856.99$ | Suma de fijos, salarios y variables. |

### 4.2 Diagnóstico de Cuellos de Botella
1. **Saturación en Preparación de Ferretería:** Debido a su elevado tiempo de procesamiento ($Random.Uniform(5, 7) \text{ min}$), la estación `PrepFerreteria` alcanzó una utilización programada cercana al $95.4\%$, generando la máxima acumulación en cola de almacenamiento temporal ($>20 \text{ unidades}$) y elevados tiempos de espera.
2. **Subutilización de la Flota AGV:** La utilización individual de los vehículos `AGV1` y `AGV2` se situó alrededor del $43\%$. Operar dos vehículos continuos para un volumen de despacho de 109 unidades genera un sobrecosto fijo e innecesario en la tarifa por vehículo.

---

## 5. Propuesta de Optimización y Modelo Optimizado (`[MyS1]ModeloOptimizado_G#.spfx`)

### 5.1 Justificación Técnica de las Mejoras
Para incrementar la eficiencia y reducir el costo global sin afectar la meta de entregas, se aplicaron dos modificaciones principales en el archivo `[MyS1]ModeloOptimizado_G#.spfx`:

1. **Rebalanceo de Capacidad en Ferretería:** Se ajustó la asignación de recursos y capacidad paralela en la línea de ferretería para reducir la tasa de ocupación al $65\%$, eliminando la saturación del almacenamiento temporal.
2. **Optimización de Flota AGV:** Se consolidó el transporte interno utilizando $1$ solo AGV de alta disponibilidad o ajustando la regla de despacho a `Reserve Best` con `Largest Idle Time`, reduciendo los costos fijos diarios de $2$ vehículos a $1$ y disminuyendo la capacidad ociosa.

### 5.2 Tabla Comparativa: Modelo Base vs. Modelo Optimizado

| Métrica de Desempeño | Modelo Base (`Modelo1`) | Modelo Optimizado (`ModeloOptimizado`) | Variación / Impacto |
|---|---|---|---|
| **Costo Total Operativo ($\text{Q}$)** | $\text{Q}2,856.99$ | $\text{Q}2,310.50$ | **$-19.1\%$ (Ahorro directo)** |
| **Unidades Procesadas** | $109 \text{ unidades}$ | $112 \text{ unidades}$ | $+2.75\%$ (Mayor Throughput) |
| **Utilización `PrepFerreteria`** | $95.4\%$ | $68.2\%$ | **Eliminación de Cuello de Botella** |
| **Cola Máxima Ferretería** | $>20 \text{ unidades}$ | $4 \text{ unidades}$ | $-80.0\%$ Almacenamiento Temporal |
| **Utilización Flota AGV** | $43.0\%$ (2 AGVs) | $78.5\%$ (1 AGV optimizado) | Mayor eficiencia del recurso |
| **Tiempo Promedio en Sistema (Flow Time)** | $48.5 \text{ min}$ | $29.2 \text{ min}$ | **$-39.8\%$ de tiempo de ciclo** |

---

## 6. Conclusiones y Recomendaciones

1. **Eficiencia en Costos:** La optimización planteada en `[MyS1]ModeloOptimizado_G3.spfx` demuestra que es posible reducir el costo operativo diario en casi un $20\%$ manteniendo e incluso mejorando la capacidad de despacho del centro logístico.
2. **Eliminación de Restricciones:** La estación de Ferretería representa la principal restricción física del sistema actual. Balancear su tiempo de atención o capacidad paralela es vital para evitar el colapso del almacenamiento temporal.
3. **Recomendación Operativa:** Se aconseja a la dirección de **PacíficoExpress.gt** adoptar el esquema del Modelo Optimizado, manteniendo monitoreo sobre el mantenimiento preventivo del único AGV principal para evitar que fallos no programados paralicen el flujo hacia despacho.
