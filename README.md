# PRUEBA TÉCNICA — DATA SCIENTIST / DATA INFRASTRUCTURE

## Contenido

La prueba contiene los siguientes directorios:

* `00_ENUNCIADO`: documento con el caso y los requisitos.
* `01_DATOS`: maestros, inventario e inputs diarios.
* `02_MOCK_API`: API local de tipos de cambio.

## Inicio rápido

1. Abrir el documento de enunciado.
2. En una terminal, ejecutar:

```bash
python 02_MOCK_API/run_mock_api.py
```

3. Comprobar que la API está disponible accediendo a:

```text
http://127.0.0.1:8000/health
```

4. Desarrollar la solución en una carpeta nueva llamada:

```text
solution/
```

Se permite utilizar **Internet, documentación y herramientas de IA** durante la realización de la prueba.

La solución entregada debe poder reproducirse en local **sin credenciales ni servicios de pago**.

## Volumen aproximado

* 10 ficheros diarios de reservas.
* Varios miles de filas, incluyendo versiones históricas, duplicados e incidencias.
* 30 fechas de estancia.
* 10 hoteles en el maestro.

---

## 1. Contexto

El equipo de Data recibe diariamente información de reservas desde distintos sistemas hoteleros.

Los ficheros pueden contener:

* Duplicados.
* Correcciones tardías.
* Registros inválidos.
* Importes expresados en distintas monedas.

Los importes deben convertirse a euros consultando un servicio interno de tipos de cambio.

El objetivo es construir un proceso **local, reproducible e idempotente** que:

1. Ingiera los datos.
2. Valide su calidad.
3. Mantenga el último estado conocido de cada reserva.
4. Genere una tabla diaria de métricas por hotel.

---

## 2. Material entregado

| Recurso                                | Descripción                                                |
| -------------------------------------- | ---------------------------------------------------------- |
| `01_DATOS/hotels.csv`                  | Maestro de hoteles, zona horaria y estado activo/inactivo. |
| `01_DATOS/hotel_inventory.csv`         | Habitaciones disponibles por hotel y fecha de estancia.    |
| `01_DATOS/incoming/reservations_*.csv` | Ficheros diarios de reservas.                              |
| `02_MOCK_API/run_mock_api.py`          | API local de tipos de cambio.                              |
| `02_MOCK_API/rates.json`               | Datos utilizados por la API local.                         |

---

## 3. Puesta en marcha de la API local

Ejecuta la API en una terminal independiente:

```bash
python 02_MOCK_API/run_mock_api.py
```

### Endpoints disponibles

```http
GET /health
```

```http
GET /exchange-rates?date=YYYY-MM-DD
```

La API simula un **fallo temporal la primera vez que se consulta cada fecha**.

La solución debe:

* Manejar este comportamiento mediante reintentos.
* Registrar claramente los fallos producidos durante las consultas.

---

## 4. Requisitos funcionales

### 4.1. Ingesta incremental

La solución debe:

* Leer todos los ficheros `reservations_*.csv` de la carpeta de entrada.
* Conservar una trazabilidad mínima que incluya:

  * Fichero de origen.
  * Fecha/hora de ingesta.
  * Número o identificador de ejecución.
* Evitar que una segunda ejecución duplique datos o métricas.
* Detectar registros exactamente duplicados.
* Procesar correctamente las correcciones tardías.

Para cada combinación:

```text
hotel_id + reservation_id + stay_date
```

debe prevalecer el **registro válido con el `modified_at` más reciente**.

El dataset puede contener empates de `modified_at` con valores distintos. No se prescribe una única forma de resolver estos conflictos, pero la solución adoptada debe ser:

* Determinista.
* Trazable.
* Documentada.

Un registro antiguo **nunca debe sobrescribir un estado más reciente**, aunque llegue en un fichero procesado posteriormente.

---

### 4.2. Validación y normalización

Como mínimo, deben aplicarse las siguientes reglas de calidad.

#### Campos obligatorios

Los siguientes campos no pueden estar vacíos:

* `hotel_id`
* `reservation_id`
* `stay_date`

#### Hoteles

* El `hotel_id` debe existir en el maestro de hoteles.
* El hotel debe estar marcado como activo.

#### Fechas y timestamps

* `stay_date` debe tener formato ISO:

```text
YYYY-MM-DD
```

* `created_at` debe ser un timestamp válido.
* `modified_at` debe ser un timestamp válido.

Los timestamps de entrada están expresados en la **zona horaria del hotel**.

La solución debe almacenarlos normalizados a **UTC**.

#### Estado de la reserva

`status` únicamente puede tomar uno de los siguientes valores:

```text
CONFIRMED
CANCELLED
CHECKED_IN
CHECKED_OUT
```

#### Habitaciones e ingresos

* `rooms` no puede ser negativo.
* `room_revenue` no puede ser negativo.

Una reserva con:

```text
status = CANCELLED
```

debe cumplir:

```text
rooms = 0
room_revenue = 0
```

#### Monedas

Solo se admiten monedas disponibles en la API de tipos de cambio para la fecha correspondiente a `stay_date`.

### Gestión de registros inválidos

Los registros inválidos **no deben detener el procesamiento completo del lote**.

Deben almacenarse en una tabla o fichero de cuarentena que incluya, como mínimo:

* Registro original.
* Motivo del rechazo.
* Fichero de origen.
* Identificador de ejecución.

### Calidad de los maestros

Los maestros también pueden contener:

* Duplicados.
* Combinaciones sin inventario.
* Inconsistencias puntuales.

La solución debe evitar que estos problemas:

* Dupliquen métricas.
* Falseen los resultados.

El criterio utilizado para tratar estas situaciones debe quedar documentado.

---

### 4.3. Conversión de moneda

La conversión de ingresos debe realizarse consultando la API local mediante `stay_date`.

El campo:

```text
room_revenue
```

debe convertirse a euros utilizando:

```text
rates_to_eur
```

La implementación debe cumplir los siguientes requisitos:

* Implementar **al menos dos reintentos** ante errores temporales `5xx`.
* Evitar llamadas repetidas a la API para una misma fecha dentro de una ejecución.
* Utilizar para ello una caché:

  * En memoria, o
  * Persistente.
* Si no se puede recuperar un tipo de cambio necesario, el comportamiento debe ser:

  * Explícito.
  * Trazable.
  * No silencioso.

## 4.4. Persistencia en SQLite

La solución debe utilizar **SQLite** como sistema de persistencia.

La base de datos debe permitir consultar, como mínimo:

* Registros ingeridos o su trazabilidad equivalente.
* Último estado válido de cada reserva.
* Registros rechazados y motivo del rechazo.
* Histórico de ejecuciones, incluyendo:

  * Estado.
  * Tiempos.
  * Contadores principales.
* Métricas diarias por hotel.

El modelo de datos exacto queda a criterio del candidato.

Se valorarán especialmente:

* Uso adecuado de claves.
* Restricciones de integridad.
* Índices.
* Separación de responsabilidades entre tablas.
* Facilidad de mantenimiento.
* Claridad del modelo.

---

## 4.5. Métricas diarias

La solución debe generar una tabla:

```text
daily_hotel_metrics
```

y un fichero equivalente:

```text
daily_hotel_metrics.csv
```

El resultado debe estar ordenado por:

1. `stay_date`
2. `hotel_id`

La tabla debe contener las siguientes columnas:

| Columna                  | Definición                                                                         |
| ------------------------ | ---------------------------------------------------------------------------------- |
| `stay_date`              | Fecha de estancia.                                                                 |
| `hotel_id`               | Identificador del hotel.                                                           |
| `active_reservations`    | Número de reservas distintas con estado `CONFIRMED`, `CHECKED_IN` o `CHECKED_OUT`. |
| `cancelled_reservations` | Número de reservas distintas con estado `CANCELLED`.                               |
| `active_rooms`           | Suma de `rooms` de las reservas activas.                                           |
| `available_rooms`        | Habitaciones disponibles según el maestro de inventario.                           |
| `occupancy_pct`          | `active_rooms / available_rooms × 100`.                                            |
| `revenue_eur`            | Suma de `room_revenue` convertido a EUR para las reservas activas.                 |

Se consideran reservas activas aquellas cuyo estado sea:

```text
CONFIRMED
CHECKED_IN
CHECKED_OUT
```

Si no existe inventario para una combinación `hotel_id + stay_date`, **no debe inventarse un valor**.

El candidato deberá decidir cómo representar esta situación y documentar el criterio aplicado.

---

## 4.6. Ejecución y observabilidad

La solución debe poder ejecutarse mediante **un único comando**, claramente documentado en el README.

Los siguientes parámetros deben ser configurables y no estar repartidos o hardcodeados por el código:

* Paths de entrada y salida.
* URL de la API.
* Ruta de la base de datos SQLite.
* Otros parámetros relevantes de ejecución.

La solución debe generar logs legibles que permitan identificar:

* Inicio de la ejecución.
* Fin de la ejecución.
* Errores.
* Principales contadores del proceso.

Como mínimo, deben registrarse los siguientes contadores:

* Filas leídas.
* Filas válidas.
* Filas inválidas.
* Filas duplicadas.
* Filas insertadas.
* Filas actualizadas.
* Métricas generadas.

Cuando se produzca un error no recuperable, el proceso debe finalizar con un **código de salida distinto de cero**.

---

## 4.7. Pruebas

La solución debe incluir **al menos cuatro pruebas automatizadas**.

Las pruebas deben cubrir varios de los siguientes escenarios:

* Validación de registros inválidos.
* Eliminación de duplicados exactos.
* Prevalencia de la corrección con el `modified_at` más reciente.
* Protección frente a una actualización que llega más tarde, pero cuyo `modified_at` es más antiguo.
* Idempotencia de una segunda ejecución.
* Reintento de la API ante un error temporal.
* Cálculo correcto de una métrica diaria.

El comando necesario para ejecutar las pruebas debe quedar documentado en el README.

---

# 5. Pregunta de diseño

Incluye un documento de **máximo una página** explicando cómo evolucionarías esta solución local hacia un entorno productivo.

Debes elegir una de estas alternativas:

* **AWS**
* **Microsoft Fabric / Azure**

La propuesta debe cubrir, como mínimo, los siguientes aspectos:

* Componentes principales y flujo de datos.
* Orquestación y planificación de los procesos.
* Almacenamiento de datos:

  * Raw.
  * Curados.
  * Métricas.
* Gestión de secretos y permisos.
* Monitorización, alertas, logs y reintentos.
* Despliegue, control de versiones y CI/CD.
* Estrategia para soportar un aumento significativo del volumen.
* Cómo evolucionarías la solución ante una posible necesidad de procesamiento **near real time**.

No es necesario desarrollar la arquitectura. Se evaluará principalmente la capacidad para justificar las decisiones propuestas.

---

# 6. Entregables

La entrega debe incluir:

1. **Código fuente** de la solución.
2. **README** con:

   * Requisitos.
   * Instalación.
   * Comando de ejecución.
   * Estructura del proyecto.
   * Decisiones principales.
   * Limitaciones conocidas.
   * Tiempo aproximado empleado.
3. **Base de datos SQLite** generada o instrucciones para crearla.
4. Fichero:

   ```text
   daily_hotel_metrics.csv
   ```
5. **Pruebas automatizadas** y comando para ejecutarlas.
6. **Documento breve de diseño de producción**.
7. **Fichero reproducible de dependencias**, por ejemplo:

   * `requirements.txt`
   * `pyproject.toml`
   * O equivalente.
8. **Declaración breve del uso de herramientas de IA**, si aplica.

---

# 7. Condiciones de realización

Se permite utilizar:

* Internet.
* Documentación técnica.
* Asistentes de inteligencia artificial.

El uso de estas herramientas **no se puntuará por sí mismo**.

Se evaluarán:

* El resultado entregado.
* El criterio aplicado.
* La capacidad para explicar y defender la solución.

Además, deben cumplirse las siguientes condiciones:

* Debes revisar, entender y probar todo el código entregado.
* No es necesario compartir conversaciones ni prompts utilizados.
* Si se han utilizado herramientas de IA, basta con indicar brevemente cuáles y para qué se utilizaron.
* La solución debe poder ejecutarse en local sin credenciales ni servicios externos de pago.
* Las dependencias pueden descargarse de Internet, pero deben quedar declaradas de forma reproducible.
* No se permite modificar manualmente los ficheros de entrada para corregir incidencias.
* No se permite hardcodear el resultado esperado.
* No es necesario desarrollar una interfaz gráfica.
* No se exige una solución perfecta: se priorizará que sea **ejecutable, clara, mantenible y justificable**.

---

# 8. Qué se valorará

Se valorarán especialmente los siguientes aspectos:

* Corrección funcional y consistencia de los resultados.
* Idempotencia.
* Robustez.
* Tratamiento de errores.
* Calidad del modelo de datos.
* Trazabilidad.
* Uso adecuado de Python y SQL.
* Estructura y modularidad del código.
* Calidad de las pruebas.
* Calidad de la documentación.
* Capacidad para explicar y defender las decisiones tomadas.
* Uso responsable y transparente de:

  * Documentación.
  * Internet.
  * Herramientas de inteligencia artificial.

---

## Nota final

Indica también: 

* **Tiempo aproximado empleado** en la realización de la prueba.
* **Qué mejorarías si dispusieras de dos horas adicionales**.
* **Qué herramientas de IA utilizaste y para qué**, si aplica.

