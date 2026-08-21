# Prueba técnica — Data Scientist / Data Infrastructure

## Contenido

La prueba contiene los siguientes directorios:

* `01_DATOS`: maestros, inventario e inputs diarios.
* `02_MOCK_API`: API local de tipos de cambio.

## Inicio rápido

1. Ejecutar la API local en una terminal:

```bash
python 02_MOCK_API/run_mock_api.py
```

2. Comprobar que está disponible:

```text
http://127.0.0.1:8000/health
```

3. Desarrollar la solución en una carpeta nueva:

```text
solution/
```

La solución entregada debe poder reproducirse en local **sin credenciales ni servicios externos de pago**.

## Volumen aproximado

* 10 ficheros diarios de reservas.
* Varios miles de filas, incluyendo versiones históricas, duplicados e incidencias.
* 30 fechas de estancia.
* 10 hoteles en el maestro.

---

# 1. Contexto

El equipo de Data recibe diariamente información de reservas desde distintos sistemas hoteleros.

Los ficheros pueden contener duplicados, correcciones tardías, registros inválidos e importes expresados en distintas monedas. Estos últimos deben convertirse a euros mediante un servicio interno de tipos de cambio.

El objetivo es construir un proceso **local, reproducible e idempotente** que:

1. Ingiera los datos.
2. Valide su calidad.
3. Mantenga el último estado conocido de cada reserva.
4. Genere una tabla diaria de métricas por hotel.

---

# 2. Material entregado

| Recurso                                | Descripción                                                |
| -------------------------------------- | ---------------------------------------------------------- |
| `01_DATOS/hotels.csv`                  | Maestro de hoteles, zona horaria y estado activo/inactivo. |
| `01_DATOS/hotel_inventory.csv`         | Habitaciones disponibles por hotel y fecha de estancia.    |
| `01_DATOS/incoming/reservations_*.csv` | Ficheros diarios de reservas.                              |
| `02_MOCK_API/run_mock_api.py`          | API local de tipos de cambio.                              |
| `02_MOCK_API/rates.json`               | Datos utilizados por la API local.                         |

---

# 3. Puesta en marcha de la API local

Ejecuta la API en una terminal independiente:

```bash
python 02_MOCK_API/run_mock_api.py
```

## Endpoints disponibles

```http
GET /health
```

```http
GET /exchange-rates?date=YYYY-MM-DD
```

La API simula un **fallo temporal la primera vez que se consulta cada fecha**.

La solución deberá gestionar este comportamiento mediante reintentos y registrar los errores producidos durante las consultas.

---

# 4. Requisitos funcionales

## 4.1. Ingesta incremental

La solución debe:

* Leer todos los ficheros `reservations_*.csv` de la carpeta de entrada.
* Mantener trazabilidad mínima mediante:

  * Fichero de origen.
  * Fecha/hora de ingesta.
  * Identificador de ejecución.
* Evitar que una segunda ejecución duplique datos o métricas.
* Detectar registros exactamente duplicados.
* Procesar correctamente correcciones tardías.

Para cada combinación:

```text
hotel_id + reservation_id + stay_date
```

debe prevalecer el **registro válido con el `modified_at` más reciente**.

El dataset puede contener empates de `modified_at` con valores diferentes. No se prescribe una única resolución, pero el criterio elegido deberá ser:

* Determinista.
* Trazable.
* Documentado.

Un registro antiguo **nunca debe sobrescribir un estado más reciente**, aunque llegue en un fichero procesado posteriormente.

---

## 4.2. Validación y normalización

Como mínimo, deben aplicarse las siguientes reglas.

### Campos obligatorios

No pueden estar vacíos:

* `hotel_id`
* `reservation_id`
* `stay_date`

### Hoteles

* El `hotel_id` debe existir en el maestro.
* El hotel debe estar activo.

### Fechas y timestamps

* `stay_date` debe tener formato ISO `YYYY-MM-DD`.
* `created_at` y `modified_at` deben ser timestamps válidos.
* Los timestamps de entrada están expresados en la zona horaria del hotel y deben almacenarse normalizados a **UTC**.

### Estado

`status` solo puede tomar uno de estos valores:

```text
CONFIRMED
CANCELLED
CHECKED_IN
CHECKED_OUT
```

### Habitaciones e ingresos

* `rooms` no puede ser negativo.
* `room_revenue` no puede ser negativo.
* Una reserva `CANCELLED` debe tener:

```text
rooms = 0
room_revenue = 0
```

### Monedas

Solo se admiten monedas disponibles en la API para la fecha indicada en `stay_date`.

### Registros inválidos

Los registros inválidos **no deben detener el lote completo**.

Deben almacenarse en una tabla o fichero de cuarentena incluyendo, como mínimo:

* Registro original.
* Motivo del rechazo.
* Fichero de origen.
* Identificador de ejecución.

### Calidad de los maestros

Los maestros pueden contener duplicados, combinaciones sin inventario u otras inconsistencias puntuales.

La solución deberá impedir que estas incidencias dupliquen o falseen las métricas y documentar el criterio utilizado para tratarlas.

---

## 4.3. Conversión de moneda

`room_revenue` debe convertirse a euros utilizando los tipos contenidos en:

```text
rates_to_eur
```

La API deberá consultarse utilizando `stay_date`.

La implementación debe:

* Realizar **al menos dos reintentos** ante errores temporales `5xx`.
* Evitar llamadas repetidas para una misma fecha dentro de una ejecución mediante caché en memoria o persistente.
* Registrar de forma explícita cualquier tipo de cambio que no pueda recuperarse.
* Evitar comportamientos silenciosos ante errores de conversión.

---

## 4.4. Persistencia en SQLite

La solución debe utilizar **SQLite** como sistema de persistencia.

La base de datos deberá permitir consultar, como mínimo:

* Registros ingeridos o su trazabilidad equivalente.
* Último estado válido de cada reserva.
* Registros rechazados y motivo.
* Histórico de ejecuciones con estado, tiempos y contadores.
* Métricas diarias por hotel.

El modelo exacto queda a criterio del candidato.

Se valorará especialmente:

* Uso adecuado de claves y restricciones.
* Índices.
* Separación de responsabilidades.
* Claridad y facilidad de mantenimiento.

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

El resultado debe estar ordenado por `stay_date` y `hotel_id`.

| Columna                  | Definición                                                               |
| ------------------------ | ------------------------------------------------------------------------ |
| `stay_date`              | Fecha de estancia.                                                       |
| `hotel_id`               | Identificador del hotel.                                                 |
| `active_reservations`    | Reservas distintas con estado `CONFIRMED`, `CHECKED_IN` o `CHECKED_OUT`. |
| `cancelled_reservations` | Reservas distintas con estado `CANCELLED`.                               |
| `active_rooms`           | Suma de `rooms` de las reservas activas.                                 |
| `available_rooms`        | Habitaciones disponibles según el maestro de inventario.                 |
| `occupancy_pct`          | `100 x active_rooms / available_rooms`.                                  |
| `revenue_eur`            | Suma de `room_revenue` convertido a EUR para las reservas activas.       |

Se consideran reservas activas aquellas cuyo estado sea:

```text
CONFIRMED
CHECKED_IN
CHECKED_OUT
```

Si no existe inventario para una combinación `hotel_id + stay_date`, **no debe inventarse un valor**. El criterio elegido para representar esta situación deberá quedar documentado.

---

## 4.6. Ejecución y observabilidad

La solución debe poder ejecutarse mediante **un único comando**, claramente documentado.

Los principales parámetros deben ser configurables y evitar valores hardcodeados repartidos por el código. Como mínimo:

* Paths de entrada y salida.
* URL de la API.
* Ruta de la base de datos SQLite.

La ejecución debe generar logs legibles que incluyan:

* Inicio y fin del proceso.
* Errores relevantes.
* Principales contadores.

Como mínimo:

* Filas leídas.
* Filas válidas.
* Filas inválidas.
* Filas duplicadas.
* Filas insertadas.
* Filas actualizadas.
* Métricas generadas.

Ante un error no recuperable, el proceso debe finalizar con un **código de salida distinto de cero**.

---

## 4.7. Pruebas

La solución debe incluir **al menos cuatro pruebas automatizadas** que cubran varios de los siguientes escenarios:

* Validación de registros inválidos.
* Eliminación de duplicados exactos.
* Prevalencia del registro con `modified_at` más reciente.
* Protección frente a una actualización que llega posteriormente pero contiene un `modified_at` más antiguo.
* Idempotencia de una segunda ejecución.
* Reintento de la API ante un error temporal.
* Cálculo correcto de una métrica diaria.

El comando para ejecutar las pruebas deberá quedar documentado.

---

# 5. Pregunta de diseño

Incluye un documento de **máximo una página** explicando cómo evolucionarías esta solución local hacia producción.

Elige una de estas alternativas:

* **AWS**
* **Microsoft Fabric / Azure**

La propuesta deberá cubrir:

* Componentes principales y flujo de datos.
* Orquestación y planificación.
* Almacenamiento de datos raw, curados y métricas.
* Gestión de secretos y permisos.
* Monitorización, alertas, logs y reintentos.
* Despliegue, control de versiones y CI/CD.
* Estrategia ante un aumento significativo del volumen.
* Evolución hacia un posible escenario **near real time**.

No es necesario desarrollar una arquitectura exhaustiva. Se evaluará principalmente el criterio aplicado y la capacidad para justificar las decisiones.

---

# 6. Entregables

La entrega deberá incluir:

1. **Código fuente** de la solución.
2. **README** con:

   * Requisitos e instalación.
   * Comando de ejecución.
   * Estructura del proyecto.
   * Principales decisiones técnicas.
   * Limitaciones conocidas.
   * Tiempo aproximado empleado.
   * Qué mejorarías con algunas horas adicionales.
3. **Base de datos SQLite** generada o instrucciones para crearla.
4. **`daily_hotel_metrics.csv`**.
5. **Pruebas automatizadas** y comando para ejecutarlas.
6. **Documento breve de diseño de producción**.
7. **Fichero reproducible de dependencias**, como `requirements.txt`, `pyproject.toml` o equivalente.
8. **Breve declaración sobre el uso de herramientas de IA**, si aplica.

El archivo `.zip` con la solución deberá enviarse a: **[alejandra.comesana@eurostarshotelcompany.com](mailto:alejandra.comesana@eurostarshotelcompany.com)**

---

# 7. Condiciones de realización

Se permite utilizar Internet, documentación técnica y herramientas de inteligencia artificial como apoyo.

El candidato es responsable de **revisar, entender y probar todo el código entregado** y deberá ser capaz de explicar y modificar su solución durante la entrevista.

No es necesario compartir conversaciones ni prompts. Si se han utilizado herramientas de IA, bastará con indicar brevemente qué herramientas se utilizaron y para qué.

Además:

* La solución debe ejecutarse en local sin credenciales ni servicios externos de pago.
* Las dependencias pueden instalarse desde Internet, pero deben declararse de forma reproducible.
* No se permite modificar manualmente los ficheros de entrada para corregir incidencias.
* No se permite hardcodear el resultado esperado.
* No es necesario desarrollar una interfaz gráfica.

---

# 8. Presentación de la solución

El candidato presentará la solución al equipo de Data durante aproximadamente **20 minutos**.

La presentación deberá permitir entender:

* Cómo ha interpretado el problema.
* Cómo ha estructurado la solución.
* Las principales decisiones tomadas y su justificación.
* Cómo se garantiza la calidad, idempotencia y trazabilidad.
* Los problemas detectados en los datos.
* Las conclusiones o resultados más relevantes.
* Qué información adicional solicitaría.
* Qué mejoraría antes de llevar la solución a producción.

Durante la entrevista podrán solicitarse **pequeños cambios en directo** sobre la solución.

---

# 9. Qué se valorará

Se valorarán especialmente:

1. **Corrección funcional y consistencia de los resultados.**
2. **Idempotencia, robustez y tratamiento de errores.**
3. **Calidad del modelo de datos y trazabilidad.**
4. **Uso adecuado de Python y SQL.**
5. **Estructura, modularidad y mantenibilidad.**
6. **Calidad de las pruebas y documentación.**
7. **Comprensión del problema y capacidad para justificar las decisiones.**

No se espera una solución completamente preparada para producción.

En caso de no completar algún apartado, el candidato podrá explicar cómo lo habría abordado.
