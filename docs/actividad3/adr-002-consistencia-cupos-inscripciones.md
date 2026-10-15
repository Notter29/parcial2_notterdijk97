# ADR-002: Estrategia para Mantener la Consistencia de Cupos e Idempotencia ante Concurrencia Extraordinaria (Escenario B)

* **Estado:** Aprobado  
* **Fecha:** Octubre de 2026  
* **Autor:** Noah Van Otterdijk Silva - 1023030297  
* **Proyecto:** Plataforma de Inscripción Académica (PIA) - Universidad de la Salle  

---

## 1. Contexto y Problema

Durante el inicio de las jornadas de matrícula, la demanda de cupos para asignaturas de alta rotación genera una intensa contención transaccional. En condiciones normales se conectan 1.200 estudiantes simultáneos (C01), pero ante situaciones de estrés (Escenario B correspondiente a documento de identidad terminado en dígito impar), el volumen de usuarios concurrentes se incrementa en un **50% (alcanzando 1.800 estudiantes simultáneos)**.

Adicionalmente, debido a la variabilidad de la conexión de red o la ansiedad del usuario, los estudiantes ejecutan **hasta cinco (5) reintentos automáticos o clics repetidos** en ventanas de pocos segundos para la misma solicitud de inscripción.

Bajo este escenario de alta concurrencia se presentan dos riesgos críticos que han afectado al sistema durante los últimos 12 años:
1. **Sobrecupo en grupos académicos:** Si el sistema verifica el cupo y luego lo asigna en pasos no atómicos, dos o más transacciones concurrentes leen cupos disponibles y confirman la inscripción, violando la restricción estricta de capacidad física de las aulas (Condición C04).
2. **Inscripciones duplicadas y colapso de conexiones:** Si no se gestionan los reintentos, el mismo estudiante puede quedar inscrito múltiples veces en el mismo grupo o agotar el pool de conexiones del servidor de base de datos PostgreSQL mediante bloqueos mutuos (*deadlocks*).

El problema a resolver es: **¿Cómo garantizar matemáticamente cero sobrecupos y exactamente una ejecución por solicitud (idempotencia), manteniendo el 95% de las respuestas en menos de 2 segundos sin saturar el motor relacional PostgreSQL?**

---

## 2. Drivers Arquitectónicos Relacionados

* **DRV-01 (Consistencia Estricta de Cupos / Cero Sobrecupo):** Ningún grupo académico puede registrar más estudiantes que su capacidad autorizada ($\Delta \text{Cupo} = 0$, Condición C04).
* **DRV-04 (Idempotencia y Manejo de Reintentos Masivos):** Resolver el 100% de las solicitudes duplicadas ($N \le 5$) sin generar transacciones redundantes ni errores 500 (Escenario B).
* **DRV-02 (Latencia de Respuesta):** Percentil 95 ($P_{95}$) de latencia $\le 2.0$ segundos bajo concurrencia pico (Condición C02).
* **DRV-03 (Concurrencia Masiva):** Soportar 1.800 conexiones simultáneas sin denegación del servicio (Condición C01 y Escenario B).

---

## 3. Alternativas Técnicamente Viables

### Alternativa 1: Bloqueo Pesimista en Base de Datos (`SELECT ... FOR UPDATE`)
Cada vez que un estudiante intenta inscribirse, la aplicación abre una transacción en PostgreSQL y ejecuta un bloqueo pesimista explícito sobre la fila del grupo:
```sql
BEGIN;
SELECT cupos_disponibles FROM grupo_academico WHERE id_grupo = :id FOR UPDATE;
-- La aplicación valida si cupos > 0 en memoria.
UPDATE grupo_academico SET cupos_disponibles = cupos_disponibles - 1 WHERE id_grupo = :id;
INSERT INTO inscripcion (...);
COMMIT;
```

### Alternativa 2: Nivel de Aislamiento Serializable en PostgreSQL
Ejecutar las transacciones bajo el nivel de aislamiento más estricto del estándar SQL (`SERIALIZABLE` mediante SSI - *Serializable Snapshot Isolation* en PostgreSQL). El motor detecta anomalías de lectura/escritura concurrentes y aborta automáticamente las transacciones conflictivas arrojando un error de serialización (`SQLSTATE 40001`), obligando a la capa de aplicación a reintentar toda la transacción.

### Alternativa 3: Reserva Atómica Condicional a Nivel de Fila + Filtro de Idempotencia en Redis y Restricción `UNIQUE` en PostgreSQL
Combinar tres mecanismos complementarios en capas:
1. **Filtro de Idempotencia en Memoria (Edge/App):** Antes de tocar la base de datos, un interceptor registra una clave atómica en Redis (`SET idemp_key <id_estudiante> NX EX 60`). Si entran reintentos con el mismo token en esa ventana, son absorbidos devolviendo el estado en curso o la respuesta en caché.
2. **Reserva Atómica Condicional (PostgreSQL):** Se elimina el `SELECT` previo y se ejecuta una sola sentencia SQL que evalúa y decrementa el cupo atómicamente a nivel de motor relacional:
   ```sql
   UPDATE grupo_academico 
   SET cupos_disponibles = cupos_disponibles - 1,
       version = version + 1
   WHERE id_grupo = :idGrupo 
     AND cupos_disponibles > 0;
   ```
   Si el número de filas afectadas es `0`, el motor garantiza que el cupo se agotó y se hace *Rollback* inmediato.
3. **Restricción de Integridad Relacional:** La tabla `inscripcion` define una restricción `UNIQUE (id_estudiante, id_grupo, periodo)` como salvaguarda inquebrantable a nivel de motor.

---

## 4. Criterios de Comparación

1. **Riesgo de Bloqueos Mutuos (*Deadlocks*) y Contención:** Capacidad de despachar peticiones masivas sin encolar hilos de ejecución en el catálogo de cerraduras (`pg_locks`).
2. **Impacto en Latencia y Rendimiento ($P_{95} \le 2\text{ s}$):** Tiempo efectivo de retención del bloqueo y tiempo total de ciclo de la transacción.
3. **Sobrecarga de Conexiones en PostgreSQL (Uso de PgBouncer):** Consumo del pool de conexiones ante 1.800 concurrentes.
4. **Efectividad contra el Escenario B (5 Reintentos Compulsivos):** Capacidad de descartar ráfagas redundantes sin desgastar el almacenamiento en disco.

---

## 5. Matriz de Evaluación de Alternativas

| Criterio de Comparación | Alternativa 1: Bloqueo Pesimista | Alternativa 2: Aislamiento Serializable | Alternativa 3: Reserva Atómica + Idempotencia |
| :--- | :--- | :--- | :--- |
| **Contención y Deadlocks** | **Muy Deficiente:** Cuando 200 estudiantes compiten por los últimos 5 cupos de un grupo, 195 quedan encolados esperando el lock de fila. Se disparan timeouts y deadlocks en PostgreSQL. | **Deficiente:** En alta contención, más del 60% de las transacciones son abortadas con error `40001`, saturando la CPU con reintentos continuos de aplicación. | **Excelente:** Cero bloqueos de tabla. El lock a nivel de fila durante un `UPDATE` condicional dura menos de 1 milisegundo en PostgreSQL. Cero deadlocks. |
| **Latencia ($P_{95} \le 2\text{ s}$)** | **Deficiente:** Las colas de espera en base de datos elevan la latencia a 5 - 15 segundos bajo 1.800 usuarios. | **Deficiente:** La necesidad de reintentar transacciones completas abortadas dispara la latencia promedio. | **Excelente:** La sentencia atómica se resuelve en $\le 5$ ms. El 95% de las peticiones se completan en menos de 1,2 segundos. |
| **Conexiones PostgreSQL** | **Muy Deficiente:** Mantiene conexiones abiertas durante toda la lógica intermedia de la app, saturando PgBouncer. | **Deficiente:** Múltiples reintentos agotan el pool de conexiones disponibles. | **Excelente:** La transacción dura milisegundos; la conexión se toma y se libera de inmediato en PgBouncer. |
| **Mitigación Escenario B** | **Inexistente:** Los 5 reintentos golpean la base de datos como 5 transacciones independientes que compiten entre sí. | **Inexistente:** Los reintentos multiplican los conflictos de serialización en el motor. | **Excelente:** El filtro en Redis frena los 4 reintentos duplicados en memoria en $< 2$ ms, protegiendo a PostgreSQL. |

---

## 6. Decisión Seleccionada y Justificación Técnica

**Se decide adoptar la Alternativa 3: Reserva Atómica Condicional a Nivel de Fila en PostgreSQL, combinada con Filtro de Idempotencia en Redis y Restricción `UNIQUE` en Base de Datos.**

### Justificación:
1. **Garantía matemática de Cero Sobrecupo (C04):** En PostgreSQL, una sentencia `UPDATE ... WHERE cupos_disponibles > 0` adquiere una cerradura exclusiva a nivel de tupla (*Tuple Lock*) únicamente durante la ejecución de la sentencia dentro del motor relacional. Dado que la condición `cupos_disponibles > 0` se reevalúa sobre la versión más reciente de la fila (*MVCC Read Committed*), es físicamente imposible que dos transacciones decrementen el cupo si el contador llegó a cero. Si el resultado de filas afectadas es `0`, el sistema sabe con certeza determinística que no hay cupo.
2. **Mitigación frontal del Escenario B sin sobrecargar la base de datos:** El problema de los 5 clics automáticos no debe resolverse en PostgreSQL, ya que saturaría el subsistema de entrada/salida (I/O). Al colocar un interceptor `IdempotencyGuard` apoyado en Redis (`SET NX EX 60`), la primera petición adquiere la clave y pasa a la base de datos; las 4 peticiones redundantes que llegan a milisegundos de diferencia son detectadas como duplicadas en memoria RAM en menos de 2 ms, retornando un mensaje pedagógico de "Solicitud en proceso", sin consumir un solo proceso de PostgreSQL.
3. **Doble cinturón de seguridad contra duplicados:** En caso extremo de que el nodo de Redis sufriera una intermitencia momentánea, la restricción declarativa `UNIQUE (id_estudiante, id_grupo, periodo)` en PostgreSQL aborta cualquier intento de registrar dos veces al estudiante en la misma materia, garantizando la consistencia absoluta de los datos institucionales.

---

## 7. Consecuencias y Riesgos

### Consecuencias Positivas:
* **Rendimiento predecible:** Elimina las colas de contención y asegura que la latencia se mantenga holgadamente por debajo del objetivo de 2 segundos ($P_{95} \le 1,2$ s).
* **Eficiencia de recursos:** Permite que 1.800 estudiantes concurrentes operen de forma fluida con un pool reducido de 60 a 80 conexiones activas en PgBouncer, ajustándose a la infraestructura local existente (C08).
* **Consistencia de negocio:** Se elimina el problema histórico de asignar sobrecupos y tener que reubicar estudiantes en salones que no tienen capacidad física.

### Consecuencias Negativas y Desafíos:
* **Dependencia de un componente en memoria (Redis):** Se incorpora Redis como dependencia operativa en la infraestructura Linux para gestionar los tokens de idempotencia.
* **Manejo de errores específicos en cliente:** La aplicación cliente (SPA) debe estar preparada para interpretar cuando una petición retorna `0` filas actualizadas (HTTP 409 Conflict - Cupo Agotado) y cuando una petición es un reintento en curso (HTTP 200 OK con payload en proceso).

### Mitigación de Riesgos:
* **Resiliencia ante caída de Redis:** Si Redis no responde, el interceptor de la aplicación entra en modo *Fail-Open* degradado (deja pasar la petición), confiando plenamente en la restricción `UNIQUE` y el `UPDATE` condicional de PostgreSQL para mantener la integridad relacional.

---

## 8. Condiciones para Revisar esta Decisión

Esta decisión arquitectónica deberá revisarse si:
1. La universidad adopta un modelo académico de "reserva temporal con carrito de compras", donde los cupos se bloquean por un período de tiempo prolongado (ej. 15 minutos) mientras el estudiante arma su horario completo, lo cual exigiría un gestor de estados temporales con expiración automática (*TTL Leases*).
2. Se implementa un modelo de sobrecupo regulado por facultad mediante cuotas de excepción discrecionales que requieran aprobaciones en línea de coordinadores académicos antes de consolidar el cupo.
