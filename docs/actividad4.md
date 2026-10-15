# Propuesta Arquitectónica: Plataforma de Inscripción Académica (PIA)
## Universidad de la Salle
**Documento de Trabajo:** Actividad 4 - Selección y Validación de Tácticas Arquitectónicas  
**Estudiante:** Noah Van Otterdijk Silva - 1023030297  
**Escenario Asignado:** Escenario B (Último dígito impar = 7: Concurrencia +50% y hasta 5 reintentos por usuario)  
**Fecha:** Octubre de 2026  
**Versión:** 1.0  

---

## 1. Introducción y Enfoque Metodológico

Como estudiante de ingeniería de software a cargo del diseño de la PIA, la selección de tácticas arquitectónicas representa el puente entre los **atributos de calidad priorizados (Actividad 1)** y las **estructuras de componentes y contenedores (Actividad 2)**. 

Siguiendo el marco de diseño del *Software Engineering Institute* (SEI) y dando estricto cumplimiento al **Criterio 6 (calidad del razonamiento, espíritu crítico y justificación técnica frente a la realidad institucional)**, este documento no se limita a listar conceptos abstractos de catálogo. Cada táctica se selecciona para resolver un problema físico real del sistema legado de 12 años, operando bajo las restricciones de la Universidad de la Salle:
* Servidores propios **Linux y PostgreSQL On-Premise** sin contratación de servicios comerciales de nube (C08).
* Un equipo humano acotado de **cuatro desarrolladores y un administrador de infraestructura** (C10).
* Un entorno de estrés con **1.800 usuarios concurrentes y hasta 5 clics compulsivos por estudiante (Escenario B)**.
* Caídas de hasta **40 minutos en el sistema legado (C05 / Escenario A)**.

A continuación, se formulan, profundizan y validan **cuatro tácticas arquitectónicas fundamentales**, detallando su implementación técnica, sus compromisos (*trade-offs*) y los protocolos concretos para verificar su efectividad.

---

## 2. Táctica 1: Deduplicación e Idempotencia con Token Atómico en Memoria

### 2.1. Atributo de Calidad que Busca Mejorar
* **Consistencia Transaccional e Integridad de Datos (DRV-04 / Escenario B)**
* **Rendimiento y Control de Demanda (DRV-03 / C01 / Escenario B)**

### 2.2. Problema Específico que Pretende Resolver
En los períodos de matrícula de la Universidad de la Salle, la ansiedad del estudiante y la lentitud percibida de la red provocan que los usuarios presionen el botón de inscripción repetidamente (hasta 5 veces en ventanas de 1 a 3 segundos). En el sistema legado actual, cada clic genera una conexión HTTP independiente que intenta ejecutar transacciones paralelas sobre la misma materia, provocando:
* Saturación inmediata del pool de conexiones a la base de datos.
* Bloqueos mutuos (*deadlocks*) en la base de datos relacional.
* Duplicación de inscripciones o decrementos erróneos de cupo.

### 2.3. Componentes en los Cuales se Implementa
* **Cliente Web (SPA en TypeScript/React):** Generador del identificador único y bloqueo en capa de presentación.
* **Reverse Proxy (Nginx en Linux):** Filtro perimetral de limitación de tasa (*rate limiting*).
* **Componente `IdempotencyGuard` (Backend API):** Interceptor de peticiones HTTP en la capa de aplicación.
* **Almacén en Memoria (Redis Open-Source en Linux):** Repositorio de tokens atómicos con expiración corta.
* **Base de Datos PostgreSQL:** Restricción declarativa `UNIQUE (id_estudiante, id_grupo, periodo)` como última barrera.

### 2.4. Funcionamiento Técnico y Flujo de Ejecución

```
[Estudiante (Clic x5)] 
        |
        v
[1. SPA Web]: Genera Idempotency-Key (UUIDv4) en el 1er clic y deshabilita el botón en la UI
        |
        v  (5 requests HTTP POST con el mismo Idempotency-Key llegan a Nginx)
[2. Nginx]: limit_req atenúa ráfagas extremas (burst=10, nodelay)
        |
        v
[3. IdempotencyGuard en Backend]:
        |
        +---> Ejecuta en Redis: SET idemp:<UUIDv4> "IN_PROGRESS" NX EX 60
        |
        |--- Petición 1: Redis responde OK (clave adquirida)
        |    ===> Continúa hacia el servicio de inscripción y PostgreSQL.
        |
        |--- Peticiones 2, 3, 4 y 5: Redis responde nil (clave ya existe)
        |    ===> Interceptor detiene la petición en < 2 ms.
        |    ===> Retorna HTTP 200 OK: {"estado": "PROCESANDO", "mensaje": "Solicitud en curso"}
        |    ===> NINGUNA de las 4 peticiones duplicadas toca la base de datos PostgreSQL.
```

1. **Generación en Origen:** Cuando el estudiante pulsa "Inscribir", la SPA genera un token criptográfico `Idempotency-Key: <UUIDv4>` y deshabilita visualmente el botón en la pantalla.
2. **Control Perimetral en Nginx:** El servidor Nginx aplica la directiva `limit_req zone=inscripcion_zone burst=10 nodelay;` por IP institucional, absorbiendo ráfagas anómalas de tráfico abusivo.
3. **Adquisición Atómica en Redis:** El componente `IdempotencyGuard` intercepta la cabecera `Idempotency-Key`. Ejecuta el comando nativo atómico de Redis:
   ```text
   SET idemp:<estudiante_id>:<grupo_id>:<periodo> "IN_PROGRESS" NX EX 60
   ```
   * El modificador `NX` garantiza que la clave solo se escriba si **NO** existe previamente (*Set if Not eXists*).
   * El parámetro `EX 60` establece una expiración automática (*TTL*) de 60 segundos para evitar fugas de memoria.
4. **Despacho Diferenciado:**
   * La **primera petición** obtiene `OK` y fluye hacia la transacción de base de datos.
   * Las **peticiones 2 a 5** reciben `nil` de Redis. El interceptor interrumpe el flujo de inmediato sin tocar PostgreSQL, retornando una respuesta estándar de proceso en curso con código HTTP 200.
5. **Consolidación del Resultado:** Cuando la transacción en PostgreSQL finaliza (éxito o cupo agotado), el backend actualiza el valor de la clave en Redis con la respuesta definitiva (`SET ... "COMPLETED:PAYLOAD"`). Cualquier consulta posterior con ese mismo token devuelve el resultado almacenado en caché.

### 2.5. Beneficios y Costos Asociados

| Dimensión | Análisis Técnico para la Universidad de la Salle |
| :--- | :--- |
| **Beneficios Técnicos** | • Elimina el 80% de las consultas de escritura redundantes en PostgreSQL durante el Escenario B.<br>• Garantiza exactamente una ejecución semántica (*exactly-once*) a nivel de negocio.<br>• Tiempo de respuesta para los 4 reintentos duplicados: $< 2$ milisegundos en memoria RAM. |
| **Costos y Sobrecarga** | • Requiere mantener el servicio Redis en el servidor Linux (aproximadamente 256 MB de RAM para 100.000 tokens concurrentes con TTL de 60s).<br>• El cliente web debe gestionar estados asíncronos y cabeceras personalizadas. |
| **Compromiso (Trade-off)** | Sacrificamos una pequeña fracción de memoria volátil en Linux a cambio de blindar por completo la CPU y el disco de PostgreSQL contra la ansiedad de 1.800 estudiantes. |

### 2.6. Mecanismo para Verificar su Efectividad

* **Herramienta de Prueba:** Script de pruebas de carga automatizadas con **k6** (open-source en Linux).
* **Escenario de Prueba de Carga:**
  * Simular 300 estudiantes virtuales concurrentes durante una ventana de 10 segundos.
  * Cada estudiante virtual dispara deliberadamente **5 peticiones HTTP POST simultáneas** con el mismo `Idempotency-Key` dirigidas al mismo grupo académico (total: 1.500 peticiones en 2 segundos).
* **Métricas y Criterios de Éxito Cuantificables:**
  1. **Tasa de deduplicación:** Exactamente 300 peticiones deben llegar al motor de PostgreSQL; 1.200 peticiones deben ser resueltas por el interceptor de Redis en menos de 5 ms.
  2. **Registros duplicados:** En la tabla `inscripcion`, la consulta SQL `SELECT id_estudiante, id_grupo, COUNT(*) FROM inscripcion GROUP BY id_estudiante, id_grupo HAVING COUNT(*) > 1;` debe retornar **0 filas**.
  3. **Estabilidad de conexiones:** Las conexiones activas en `pg_stat_activity` no deben superar las 60 conexiones simultáneas administradas por PgBouncer.

---

## 3. Táctica 2: Reserva Atómica Condicional a Nivel de Fila

### 3.1. Atributo de Calidad que Busca Mejorar
* **Consistencia e Integridad Transaccional (DRV-01 / C04 - Cero Sobrecupo)**
* **Rendimiento y Capacidad de Throughput (DRV-02 / C02 / DRV-03)**

### 3.2. Problema Específico que Pretende Resolver
Cuando quedan los últimos 5 cupos de una materia de alta demanda y 200 estudiantes intentan inscribirla en el mismo segundo:
* Si el sistema usa un patrón tradicional `SELECT cupos` $\rightarrow$ `evalúa en memoria` $\rightarrow$ `UPDATE cupos`, se producen condiciones de carrera (*Race Conditions*) y sobrecupos masivos (violando la capacidad física de las aulas, C04).
* Si el sistema usa bloqueos pesimistas (`SELECT ... FOR UPDATE`), 195 estudiantes quedan esperando en colas de bloqueos de PostgreSQL (`pg_locks`), provocando agotamiento del pool de conexiones, bloqueos mutuos (*deadlocks*) y latencias superiores a 10 segundos.

### 3.3. Componentes en los Cuales se Implementa
* **Componente `GestorCupos` (Backend API):** Genera la sentencia SQL de actualización condicional y evalúa el retorno atómico.
* **Componente `InscripcionRepository`:** Ejecuta la inserción relacional de la matrícula dentro de la misma transacción.
* **Motor Relacional PostgreSQL 16 (Nodo Primario en Linux):** Aplica aislamiento *Read Committed* con cerrojos a nivel de tupla (*Tuple Locks*).

### 3.4. Funcionamiento Técnico y Flujo de Ejecución

```
[Transacción de Inscripción en PostgreSQL (Read Committed)]
                           |
                           v
   UPDATE grupo_academico 
   SET cupos_disponibles = cupos_disponibles - 1,
       version = version + 1
   WHERE id_grupo = :idGrupo 
     AND cupos_disponibles > 0;
                           |
            +--------------+--------------+
            |                             |
    [Filas Afectadas == 1]        [Filas Afectadas == 0]
            |                             |
    Cupo reservado con éxito      El cupo YA se agotó en otra tx
            |                             |
    INSERT INTO inscripcion ...   ROLLBACK INMEDIATO
    INSERT INTO outbox ...        Retorna: HTTP 409 (Sin cupo)
    COMMIT (Duración: < 2 ms)     (Duración: < 1 ms, sin locks)
```

1. **Aprovechamiento del Motor MVCC de PostgreSQL:**
   En PostgreSQL, una sentencia `UPDATE` atómica adquiere una cerradura exclusiva (*Exclusive Tuple Lock*) sobre la fila específica del grupo académico únicamente durante la ejecución de esa instrucción.
2. **Reevaluación Condicional en la Fila más Reciente:**
   Bajo el nivel de aislamiento estándar `Read Committed`, cuando múltiples transacciones intentan actualizar la misma fila simultáneamente, PostgreSQL bloquea a las transacciones posteriores hasta que la primera hace `COMMIT`. Tan pronto se libera la fila, PostgreSQL **reevalúa la cláusula `WHERE` sobre la versión recién confirmada de la tupla**.
3. **Decisión Binaria Determinística:**
   * Si tras la actualización de las transacciones previas todavía se cumple `cupos_disponibles > 0`, la sentencia decrementa el valor en 1 y retorna `Rows Affected = 1`.
   * Si el cupo llegó a 0, la cláusula `AND cupos_disponibles > 0` ya no se cumple; la sentencia **no modifica la fila** y retorna de inmediato `Rows Affected = 0`.
4. **Rollback sin Contención:**
   Al detectar 0 filas afectadas, el `GestorCupos` aborta la transacción inmediatamente con un `ROLLBACK`, liberando la conexión de base de datos a PgBouncer en menos de 1 milisegundo y arrojando una excepción controlada de negocio (`CupoAgotadoException`).

### 3.5. Beneficios y Costos Asociados

| Dimensión | Análisis Técnico para la Universidad de la Salle |
| :--- | :--- |
| **Beneficios Técnicos** | • **Garantía absoluta de Cero Sobrecupo ($\Delta \text{Cupo} = 0$):** Imposible superar la capacidad física del aula.<br>• **Cero bloqueos de tabla:** El bloqueo dura microsegundos y está confinado a una sola tupla en memoria compartida.<br>• **Eliminación total de deadlocks** en la gestión de cupos. |
| **Costos y Sobrecarga** | • Requiere que la aplicación interprete el resultado del número de filas afectadas en lugar de consultar previamente.<br>• No permite por sí sola reservas temporales tipo "apartar cupo por 15 minutos en un carrito". |
| **Compromiso (Trade-off)** | Preferimos un modelo de reserva determinística inmediata en lugar de reservas provisionales complejas, reduciendo la carga operativa para los 4 desarrolladores (C10) y garantizando la equidad de acceso para los estudiantes. |

### 3.6. Mecanismo para Verificar su Efectividad

* **Herramienta de Prueba:** Test de estrés de concurrencia extrema con **k6** ejecutado desde un nodo de prueba en Linux.
* **Escenario de Prueba de Concurrencia:**
  * Crear un grupo académico de prueba con **exactamente 5 cupos disponibles** (`cupos_disponibles = 5`).
  * Lanzar **200 hilos concurrentes simultáneos** intentando inscribir esa misma asignatura en el mismo milisegundo.
* **Métricas y Criterios de Éxito Cuantificables:**
  1. **Cero sobrecupo verificado en base de datos:**
     ```sql
     SELECT cupos_disponibles, 
            (SELECT COUNT(*) FROM inscripcion WHERE id_grupo = 'TEST-01') AS total_inscritos
     FROM grupo_academico WHERE id_grupo = 'TEST-01';
     ```
     **Resultado requerido:** `total_inscritos == 5` y `cupos_disponibles == 0`. Bajo ninguna circunstancia `total_inscritos > 5`.
  2. **Respuestas HTTP esperadas:** Exactamente 5 peticiones deben recibir HTTP 200/201 (Inscrito exitoso) y exactamente 195 peticiones deben recibir HTTP 409 Conflict (Cupo Agotado).
  3. **Tasa de Deadlocks:** La métrica `deadlocks` en la vista `pg_stat_database` debe mantenerse en **0**.

---

## 4. Táctica 3: Aislamiento Asíncrono mediante Transactional Outbox y Circuit Breaker

### 4.1. Atributo de Calidad que Busca Mejorar
* **Disponibilidad y Resiliencia del Sistema (DRV-05 / C05 / Escenario A / DRV-06 / C03)**
* **Recuperabilidad y Cero Pérdida de Datos (DRV-07 / C06 / C07)**

### 4.2. Problema Específico que Pretende Resolver
El sistema legado de 12 años presenta **interrupciones de hasta 40 minutos (C05 / Escenario A)**. Si la PIA intenta sincronizar las inscripciones de forma síncrona (bloqueante) mediante HTTP o conexiones directas de base de datos hacia el legado durante la matrícula:
* La indisponibilidad del legado tumba de inmediato la PIA, violando el acuerdo institucional de disponibilidad mensual del 99,9% (C03).
* Las conexiones web de los estudiantes se quedan colgadas esperando timeouts, agotando el pool de conexiones y provocando errores 504 Gateway Timeout.
* Se generan inconsistencias críticas donde la materia queda inscrita en la PIA pero se pierde en el camino hacia el sistema legado.

### 4.3. Componentes en los Cuales se Implementa
* **Componente `OutboxPublisher` (Backend API):** Inserta eventos en PostgreSQL dentro de la transacción local.
* **Tabla `outbox_inscripciones` en PostgreSQL:** Almacén persistente local de eventos pendientes.
* **Worker de Sincronización (`OutboxWorker`):** Servicio daemon independiente en Linux gestionado por `systemd`.
* **Módulo `CircuitBreaker` (Librería en el Worker):** Interruptor automático de protección contra fallas del legado.

### 4.4. Funcionamiento Técnico y Flujo de Ejecución

```
[Transacción Local en PIA]
   |
   +---> Reserva Cupo en PostgreSQL
   +---> Inserta Inscripción (Estado: 'PENDIENTE_SINC')
   +---> Inserta Evento en 'outbox_inscripciones' (Estado: 'PENDIENTE')
   |
[COMMIT LOCAL EN POSTGRESQL] ===> Estudiante recibe confirmación inmediata (< 200 ms)
                                  "Cupo garantizado; sincronización oficial en curso"

                                     |
               (Proceso en segundo plano desacoplado en Linux)
                                     v
                       [Worker de Sincronización]
                                     |
                                     +---> Consulta eventos pendientes:
                                     |     SELECT * FROM outbox_inscripciones 
                                     |     WHERE estado = 'PENDIENTE' 
                                     |     ORDER BY fecha_creacion ASC 
                                     |     LIMIT 50 FOR UPDATE SKIP LOCKED;
                                     |
                                     v
                           [Circuit Breaker]
                                     |
                    +----------------+----------------+
                    |                                 |
           [Estado: CLOSED (Legado OK)]      [Estado: OPEN (Legado Caído 40 min)]
                    |                                 |
        Envía HTTP/REST al Legado         Detiene llamadas al legado.
        Recibe 200 OK del Legado          No agota sockets ni memoria.
        Marca evento como 'SINCRONIZADO'  Eventos se acumulan seguros en PG.
                                          Reintenta en 60s (Half-Open).
```

1. **Garantía Atómica Dual (Local):** La inscripción del estudiante y la orden de sincronización se escriben en la misma transacción relacional de PostgreSQL. Es físicamente imposible que una inscripción exista sin su correspondiente evento Outbox.
2. **Procesamiento Concurrente sin Bloqueos (`SKIP LOCKED`):**
   El worker en segundo plano ejecuta:
   ```sql
   SELECT id_evento, payload 
   FROM outbox_inscripciones 
   WHERE estado = 'PENDIENTE' 
   ORDER BY fecha_creacion ASC 
   LIMIT 50 
   FOR UPDATE SKIP LOCKED;
   ```
   La directiva `SKIP LOCKED` permite que, si el administrador despliega múltiples hilos de worker, ninguno bloquee al otro ni compitan por las mismas filas.
3. **Comportamiento ante la Caída de 40 Minutos del Legado (Escenario A):**
   * Tras fallar 3 llamadas consecutivas hacia el legado con *timeout* superior a 2 segundos, el **Circuit Breaker entra en estado `OPEN`**.
   * Durante los 40 minutos de indisponibilidad externa, el worker **no realiza llamadas de red inútiles**.
   * La PIA continúa funcionando al 100% para los estudiantes: inscribe, valida cupos y acumula los eventos de forma segura en la base de datos local.
   * A los 40 minutos, cuando el sistema legado regresa a operación, el Circuit Breaker entra en estado `HALF-OPEN` enviando una petición de prueba (*health probe*). Al tener éxito, pasa a `CLOSED` y el worker procesa los eventos acumulados en lotes ordenados hasta vaciar la cola.

### 4.5. Beneficios y Costos Asociados

| Dimensión | Análisis Técnico para la Universidad de la Salle |
| :--- | :--- |
| **Beneficios Técnicos** | • **Disponibilidad blindada (C03):** La PIA mantiene el 99,9% de disponibilidad independientemente de si el legado se cae 40 minutos.<br>• **Cero pérdida de datos (C07 / RPO $\le$ 5 min):** Los datos residen en almacenamiento persistente de PostgreSQL.<br>• **Elimina transacciones distribuidas complejas** (2PC o Sagas orquestadas externas). |
| **Costos y Sobrecarga** | • **Consistencia eventual:** Durante la caída del legado, existe una ventana de hasta 40 minutos donde la inscripción es oficial en la PIA pero aún no figura en el historial legado.<br>• Crecimiento temporal del tamaño de la tabla `outbox_inscripciones` en disco. |
| **Compromiso (Trade-off)** | Aceptamos consistencia eventual temporal entre ambos sistemas a cambio de garantizar la continuidad absoluta del proceso de matrícula de los 24.000 estudiantes. |

### 4.6. Mecanismo para Verificar su Efectividad

* **Herramienta de Prueba:** Simulación de fallas mediante inyección de fallas en Linux con un servidor Mock del sistema legado y script de k6.
* **Escenario de Prueba de Resiliencia:**
  1. Iniciar un proceso de matrícula continuo a un ritmo de 50 inscripciones por minuto.
  2. A los 2 minutos, apagar abruptamente el mock del sistema legado durante **40 minutos continuos** (simulando C05 / Escenario A).
  3. Verificar que los estudiantes sigan matriculando normalmente en la PIA.
  4. Encender nuevamente el mock del sistema legado y monitorear la reconciliación.
* **Métricas y Criterios de Éxito Cuantificables:**
  1. **Disponibilidad percibida:** El 100% de las inscripciones enviadas a la PIA durante los 40 minutos de corte deben responder HTTP 200 con su comprobante de pre-matrícula emitido. Tasa de errores HTTP 5xx: **0%**.
  2. **Acumulación de eventos:** Al minuto 39 de la caída, la consulta `SELECT COUNT(*) FROM outbox_inscripciones WHERE estado = 'PENDIENTE'` debe contabilizar exactamente las inscripciones ocurridas durante la ventana.
  3. **Tiempo de vaciado de cola:** Tras reiniciar el legado, el 100% de los eventos acumulados deben quedar en estado `'SINCRONIZADO'` en menos de **15 minutos**, con cero discrepancias entre ambos repositorios.

---

## 5. Táctica 4: Desacoplamiento de Lecturas mediante CQRS Ligero y Caché de Catálogo

### 5.1. Atributo de Calidad que Busca Mejorar
* **Rendimiento y Latencia de Consulta (DRV-02 / C02 - $P_{95} \le 2,0$ s)**
* **Escalabilidad ante Concurrencia Masiva (DRV-03 / C01 / Escenario B - 1.800 usuarios)**

### 5.2. Problema Específico que Pretende Resolver
En la plataforma de matrícula, por cada transacción de inscripción real se ejecutan entre 8 y 12 consultas de visualización de oferta académica, asignaturas, horarios y cupos. Si los 1.800 estudiantes concurrentes ejecutan consultas pesadas (con múltiples `JOIN` entre asignaturas, docentes y aulas) directamente sobre el nodo primario de base de datos donde se hacen las inscripciones, se satura el ancho de banda del disco, se agota la memoria compartida de PostgreSQL (*shared buffers*) y la latencia de consulta se degrada a más de 6 segundos, violando la condición C02.

### 5.3. Componentes en los Cuales se Implementa
* **Servidor Nginx (Proxy Inverso):** Micro-caché perimetral de 2 segundos para catálogos estáticos.
* **Instancia Redis en Linux:** Almacenamiento en caché de la oferta académica consolidada en formato JSON.
* **PostgreSQL Standby (Réplica de Lectura en Linux):** Atiende consultas complejas que sufran *cache miss*.
* **PostgreSQL Primario:** Queda 100% reservado para transacciones de escritura (inscripciones y outbox).

### 5.4. Funcionamiento Técnico y Flujo de Ejecución

```
[Estudiante Consulta Asignaturas]
               |
               v
       [Nginx Reverse Proxy]
               | (Cache-Control: public, max-age=2)
               v
      [Módulo de Catálogo PIA]
               |
               +---> 1. Consulta Redis: GET catalogo:periodo_2026_2:facultad_ing
               |
               |--- Cache Hit (> 85% de las veces):
               |    Retorna JSON comprimido en < 15 milisegundos.
               |
               |--- Cache Miss (o expiración de TTL = 30s):
               |    +---> 2. Consulta a PostgreSQL STANDBY (Réplica de Lectura)
               |    |        SELECT ... FROM vista_oferta_academica;
               |    |
               |    +---> 3. Guarda resultado en Redis (SETEX ... 30s)
               |    +---> 4. Retorna respuesta al estudiante.
               |
[PostgreSQL PRIMARIO]: Cero consultas de lectura de catálogo. 
                       100% dedicado a reserva de cupos e inscripciones.
```

1. **Estrategia de Caché Multinivel (*Cache-Aside con TTL Corto*):**
   * La oferta de asignaturas y horarios no cambia cada segundo; cambia la disponibilidad de cupos.
   * La información del catálogo se guarda en Redis con un tiempo de vida (*TTL*) de 30 segundos.
   * Cuando un estudiante navega la oferta, la información se recupera de la memoria RAM de Redis en menos de 15 ms, sin tocar ningún disco magnético o SSD.
2. **Derivación de Consultas a la Réplica Standby:**
   * Si una consulta personalizada (ej. filtros avanzados por docente o palabras clave) no se encuentra en caché, la aplicación dirige la consulta a la **Réplica Standby de PostgreSQL**, alimentada continuamente por *Streaming Replication* de registros WAL.
   * Esto garantiza que el nodo primario de PostgreSQL mantenga su CPU y su pool de conexiones libres exclusivamente para el procesamiento transaccional de cupos.

### 5.5. Beneficios y Costos Asociados

| Dimensión | Análisis Técnico para la Universidad de la Salle |
| :--- | :--- |
| **Beneficios Técnicos** | • **Latencia ultra-baja garantizada (C02):** El 95% de las consultas se entregan en $< 200$ ms, superando holgadamente la meta de 2,0 segundos.<br>• **Protección del nodo primario:** Reduce en un 90% la carga de lectura en el servidor transaccional de base de datos.<br>• **Resiliencia:** Si el nodo primario falla, las consultas de catálogo continúan funcionando desde la réplica y la caché. |
| **Costos y Sobrecarga** | • **Ventana de eventualidad en cupos mostrados:** El cupo mostrado en el catálogo puede tener un desfase de visualización de hasta 30 segundos respecto a la base de datos transaccional.<br>• Requiere configurar y monitorear la replicación nativa por streaming en Linux. |
| **Compromiso (Trade-off)** | Asumimos que el estudiante vea un cupo estimado en pantalla de consulta, ya que **la validación definitiva y atómica de si el cupo existe se realiza con rigor estricto al momento exacto de la inscripción** (Táctica 2). |

### 5.6. Mecanismo para Verificar su Efectividad

* **Herramienta de Prueba:** Prueba de carga con **k6** simulando navegación intensiva.
* **Escenario de Prueba de Rendimiento:**
  * Simular 1.800 estudiantes virtuales navegando simultáneamente la oferta académica de las 5 facultades durante 15 minutos continuos.
* **Métricas y Criterios de Éxito Cuantificables:**
  1. **Latencia Percentil 95 ($P_{95}$):**
     * El reporte consolidado de k6 debe certificar que el $P_{95}$ para las rutas `/api/v1/catalogo/*` es **inferior a 250 milisegundos** (cumpliendo con creces la meta de $\le 2.0$ segundos de C02).
  2. **Tasa de Acierto de Caché (*Cache Hit Ratio*):**
     * Mediante el comando `INFO stats` en Redis, la métrica `keyspace_hits / (keyspace_hits + keyspace_misses)` debe ser **superior al 85%**.
  3. **Consumo de CPU en Servidores de Base de Datos:**
     * En el nodo Primario de PostgreSQL, la utilización de CPU atribuible a sentencias `SELECT` de catálogo debe ser **menor al 5%**.

---

## 6. Matriz de Cobertura: Tácticas vs. Drivers Priorizados

La siguiente matriz evidencia la trazabilidad total entre los drivers de la Actividad 1 y las tácticas seleccionadas:

| Driver Priorizado | Condición | Táctica Arquitectónica que lo Satisface | Mecanismo Clave de Mitigación |
| :--- | :--- | :--- | :--- |
| **DRV-01: Cero Sobrecupo** | C04 | **Táctica 2:** Reserva Atómica Condicional | `UPDATE ... WHERE cupos > 0` con Tuple Lock atómico en PostgreSQL. |
| **DRV-02: Latencia $\le 2$ s** | C02 | **Táctica 4:** CQRS Ligero y Caché Redis | Despacho de oferta desde memoria RAM ($< 15$ ms) y réplica standby. |
| **DRV-03: Concurrencia Masiva (1.800)** | C01, Escenario B | **Tácticas 1 y 4:** Idempotencia + CQRS | Absorción de carga en memoria y amortiguación mediante PgBouncer. |
| **DRV-04: Manejo de 5 Reintentos** | Escenario B | **Táctica 1:** Deduplicación Atómica | Interceptor `SET NX` en Redis; bloquea duplicados en $< 2$ ms. |
| **DRV-05: Indisponibilidad Legado 40m** | C05, Escenario A | **Táctica 3:** Transactional Outbox + Circuit Breaker | Desacoplamiento temporal; inscripciones locales continúan 100% activas. |
| **DRV-06: Disponibilidad Mensual 99,9%** | C03 | **Táctica 3:** Aislamiento Asíncrono | Ruptura de dependencia síncrona; el fallo externo no contagia a la PIA. |
| **DRV-07: RTO $\le 30$m / RPO $\le 5$m** | C06, C07 | **Táctica 3:** Persistencia Outbox en PostgreSQL | Cero transacciones en tránsito perdidas ante contingencias. |
| **DRV-08: On-Premise Linux/PG** | C08 | **Todas las Tácticas (1, 2, 3 y 4)** | Implementación 100% en Linux, PostgreSQL y Redis sin software propietario. |
| **DRV-10: Operable por 4 devs / 1 sysadmin** | C10 | **Todas las Tácticas (1, 2, 3 y 4)** | Patrones nativos relacionales y monolito modular sin orquestadores hipercomplejos. |
