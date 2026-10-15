# Propuesta Arquitectónica: Plataforma de Inscripción Académica (PIA)
## Universidad de la Salle
**Documento de Trabajo:** Actividad 1 - Identificación y Priorización de Drivers Arquitectónicos  
**Rol:** Arquitecto de Software  
**Fecha:** Octubre de 2026  
**Versión:** 1.0  

---

## 1. Contexto Operativo y Justificación Arquitectónica

La Universidad de la Salle atiende una comunidad de aproximadamente **24.000 estudiantes**, **1.100 docentes** y **cinco facultades**. Durante los períodos de matrícula semestral, la demanda de acceso al sistema se concentra críticamente en ventanas de tiempo muy reducidas. Actualmente, los procesos se ejecutan sobre un sistema académico desarrollado hace más de 12 años, cuya infraestructura y diseño monolítico estrechamente acoplado han alcanzado el final de su ciclo de vida efectivo.

El diagnóstico técnico del estado actual evidencia los siguientes puntos críticos que motivan el diseño de la nueva **Plataforma de Inscripción Académica (PIA)**:

1. **Saturación por concurrencia y degradación del servicio:** En las horas pico, la concurrencia de al menos 1.200 usuarios simultáneos (con incrementos imprevistos de hasta 1.800 usuarios) provoca colapsos de conexión, agotamiento del pool de conexiones a la base de datos y falta de disponibilidad de la plataforma.
2. **Reintentos descontrolados y registros duplicados:** Ante la lentitud de respuesta, los estudiantes reintentan compulsivamente la misma solicitud de inscripción (hasta 5 veces), lo que satura la capa de persistencia y genera inscripciones fantasmas, bloqueos mutuos (*deadlocks*) o duplicaciones.
3. **Inconsistencia severa en cupos académicos:** Fallas en el control de transacciones concurrentes originan sobrecupos que violan las capacidades físicas de las aulas y la planificación docente institucional.
4. **Acoplamiento nocivo con el sistema legado:** Cada validación de prerrequisitos invoca directamente el motor del sistema legado, el cual presenta interrupciones operativas no planificadas de hasta 40 minutos, trasladando su indisponibilidad directamente al proceso de matrícula.
5. **Opacidad operativa y déficit de auditoría:** No existe trazabilidad granular que permita reconstruir quién, cuándo, desde qué IP y bajo qué autorización se alteró o canceló una inscripción.
6. **Rigidez funcional ante nuevas políticas académicas:** La lógica de prerrequisitos, correquisitos y restricciones de créditos está dispersa e incrustada en procedimientos almacenados y código legado, demandando semanas de esfuerzo técnico para cambios normativos menores.

Bajo este panorama, la PIA no se plantea como un simple reemplazo cosmético de interfaz, sino como un **sistema desacoplado de alta resiliencia y alta concurrencia**, capaz de orquestar el período de matrícula de manera autónoma, blindando al estudiante de las fallas del sistema legado y garantizando la consistencia absoluta de los cupos y la sincronización fidedigna de los datos oficiales.

---

## 2. Objetivos de Negocio (Business Goals)

Los objetivos de negocio articulan el propósito estratégico de la universidad con los requerimientos que debe satisfacer la arquitectura:

* **ON-01: Garantizar la continuidad del servicio de matrícula y la equidad en el acceso.**  
  Asegurar que los 24.000 estudiantes de la universidad completen su proceso de inscripción sin interrupciones del servicio, tiempos de espera frustrantes o pérdida de turnos de matrícula, eliminando la asimetría generada por fallos tecnológicos durante las horas pico de inscripción.
* **ON-02: Preservar la integridad de la oferta académica y eliminar sobrecupos.**  
  Proteger la viabilidad administrativa y docente de las cinco facultades garantizando que ningún grupo académico exceda el 100% de su capacidad autorizada (cero sobrecupos), evitando reprocesos de asignación manual de salones y reajustes extemporáneos.
* **ON-03: Reducir los costos operativos de soporte y la dependencia directa del sistema legado durante contingencias.**  
  Permitir que la institución mantenga la ventana de inscripciones habilitada aun durante ventanas de indisponibilidad del sistema legado de hasta 40 minutos, reduciendo en al menos un 80% los incidentes escalados a mesa de ayuda técnica por bloqueos de plataforma o inscripciones inconsistentes.
* **ON-04: Facilitar la adaptabilidad normativa con el equipo técnico institucional existente.**  
  Permitir que el equipo interno de mantenimiento (4 desarrolladores y 1 administrador de infraestructura) incorpore o modifique reglas de negocio académicas sin requerir reescrituras complejas ni comprometer la estabilidad del sistema legado.

---

## 3. Restricciones Arquitectónicas (Constraints)

Las restricciones delimitan el espacio de solución tecnológica y de diseño, y son de obligatorio cumplimiento:

* **CON-01 (Ecosistema tecnológico y presupuesto): Infraestructura Linux y PostgreSQL On-Premise.**  
  La solución debe ejecutarse estrictamente sobre servidores Linux y motores de bases de datos relacionales PostgreSQL existentes en la infraestructura de la universidad. Inicialmente, la institución prohíbe la contratación de servicios comerciales adicionales (SaaS propietarios, nubes públicas comerciales de pago recurrente o licencias empresariales de terceros).
* **CON-02 (Coexistencia con Sistema Legado): Sistema legado como fuente oficial de la verdad histórica.**  
  El sistema de más de 12 años permanece como el repositorio maestro e indiscutible del historial académico previo y la definición oficial de planes de estudio y prerrequisitos. La PIA administra y consolida las inscripciones del período en curso y es responsable de su sincronización fidedigna hacia el legado, conviviendo con caídas de este de hasta 40 minutos.
* **CON-03 (Capacidad del equipo de soporte y mantenimiento): Gobernanza técnica reducida.**  
  La arquitectura, el despliegue y las tecnologías elegidas deben poder ser operadas, monitoreadas y mantenidas de manera sostenible por un equipo conformado exclusivamente por cuatro (4) desarrolladores de software y un (1) administrador de infraestructura de sistemas.
* **CON-04 (Marco Regulatorio de Protección de Datos y Trazabilidad): Cumplimiento de Ley 1581 / Habeas Data.**  
  El almacenamiento y tratamiento de información académica y personal debe contemplar mecanismos estrictos de confidencialidad, controles de acceso por roles (RBAC) y registros inmutables de auditoría técnica y funcional para soportar revisiones disciplinarias y regulatorias.

---

## 4. Drivers de Atributos de Calidad (Quality Attributes)

A partir de las condiciones de operación (C01 a C10) y los escenarios de estrés del caso, se identifican y formalizan los siguientes atributos de calidad clave:

### 4.1. Rendimiento y Eficiencia de Tiempo (Performance)
* **Demanda:** Soportar un mínimo de 1.200 usuarios concurrentes simultáneos durante las primeras horas de matrícula (C01), con capacidad de responder adecuadamente ante picos extraordinarios de hasta 1.800 concurrentes (+50%, Escenario B).
* **Latencia objetivo:** El 95% de las consultas de catálogo de asignaturas y grupos disponibles debe responder en un tiempo inferior o igual a 2 segundos (C02), independientemente del volumen de tráfico concurrente.

### 4.2. Disponibilidad y Tolerancia a Fallos (Availability & Fault Tolerance)
* **Disponibilidad global:** Mantener un uptime mensual no inferior al 99,9% durante los períodos de matrícula (C03). Esto implica un tiempo máximo de indisponibilidad acumulada no programada inferior a 43,8 minutos al mes.
* **Aislamiento de fallos externos:** Resiliencia total ante interrupciones de hasta 40 minutos en el sistema académico legado (C05 y Escenario A), garantizando que las operaciones locales prioritarias de consulta e intención de matrícula no queden bloqueadas.
* **Recuperabilidad ante desastres:** 
  * **RTO (Recovery Time Objective):** Máximo 30 minutos para restablecer el servicio completo ante fallas graves de servidor o base de datos (C06).
  * **RPO (Recovery Point Objective):** Pérdida máxima admisible de datos de 5 minutos de transacciones efectivas en caso de caída catastrófica (C07).

### 4.3. Consistencia e Integridad Transaccional (Data Consistency & Integrity)
* **Regla estricta de capacidad:** Prohibición absoluta de sobrecupo (C04). Bajo ningún escenario de concurrencia extrema o accesos simultáneos a milisegundos de diferencia, el número de inscritos en un grupo puede superar la capacidad máxima autorizada.
* **Idempotencia de transacciones:** Inmunidad frente a reintentos compulsivos de usuario (hasta 5 solicitudes idénticas originadas por el mismo estudiante en segundos, Escenario B). El sistema debe procesar la solicitud exactamente una vez (*exactly-once semantic* a nivel de negocio), sin duplicar registros ni decrementar cupos adicionales.

### 4.4. Seguridad y Auditabilidad (Security & Auditability)
* **Seguridad de datos:** Control de acceso granular basado en roles (estudiante, docente, administrador, auditor) para proteger datos sensibles de estudiantes y notas (C09).
* **Trazabilidad estricta e inmutable:** Registro pormenorizado de cada operación académica (inscripción, cancelación, reintento, confirmación de sincronización) con marca de tiempo precisa, IP de origen, usuario autenticado, estado anterior y estado resultante (C09).

### 4.5. Mantenibilidad y Modificabilidad (Maintainability & Modifiability)
* **Desacoplamiento de reglas de negocio:** Capacidad de actualizar o parametrizar reglas de inscripción (topes de créditos, excepciones de prerrequisitos, turnos de matrícula) en menos de 2 días hábiles y sin requerir reconstrucciones completas de la base de datos o el sistema legado (C10).
* **Simplicidad operativa:** La arquitectura debe evitar la sobreingeniería para ajustarse a las competencias de un equipo de 4 desarrolladores y 1 administrador de infraestructura (C10).

---

## 5. Matriz Integral de Clasificación y Priorización de Drivers

La siguiente matriz recopila y clasifica los drivers arquitectónicos. La priorización se evalúa considerando dos dimensiones críticas:
* **Importancia para el Negocio (IN):** Valor que aporta a la continuidad académica y satisfacción institucional (Alta, Media, Baja).
* **Impacto en la Arquitectura (IA):** Complejidad técnica, riesgo estructural e influencia sobre las decisiones de diseño fundamentales (Alto, Medio, Bajo).

| ID | Driver Arquitectónico | Origen | Categoría | Prioridad Global (IN / IA) | Medida de Aceptación Cuantificable | Justificación Técnica y de Negocio |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **DRV-01** | Consistencia Estricta de Cupos (Cero Sobrecupo) | C04, Reglas Académicas | Integridad / Consistencia | **Crítica (Alta / Alto)** | $\Delta \text{Cupo} = 0$ sobrecupos permitidos. En el 100% de las inscripciones: $\text{Inscritos} \le \text{Capacidad}$. | Un solo sobrecupo genera problemas de espacio físico, conflictos docentes y vicios de nulidad en la matrícula. Es una regla de negocio no negociable. |
| **DRV-02** | Latencia en Consultas de Asignaturas | C02 | Rendimiento | **Alta (Alta / Medio)** | Percentil 95 ($P_{95}$) de latencia $\le 2.0$ segundos bajo carga nominal y pico. | La consulta de oferta es la operación más frecuente (ratio 10:1 respecto a la escritura). Si la consulta es lenta, los estudiantes no pueden tomar decisiones y aumentan los reintentos. |
| **DRV-03** | Concurrencia Masiva y Escalabilidad Local | C01, Escenario B | Rendimiento / Capacidad | **Crítica (Alta / Alto)** | Soportar 1.200 usuarios concurrentes estables y absorber picos de hasta 1.800 concurrentes (+50%) sin denegación del servicio ni caída del servidor. | Evita el colapso inicial del sistema en los primeros 60 minutos de matrícula, momento en que se define la percepción de éxito institucional del nuevo sistema. |
| **DRV-04** | Idempotencia y Protección ante Reintentos | Problemas detectados, Escenario B | Integridad / Rendimiento | **Alta (Alta / Alto)** | $100\%$ de solicitudes duplicadas ($N \le 5$) identificadas y resueltas con una única transacción real, respondiendo la misma respuesta sin error 500. | Evita la saturación artificial de la base de datos y la generación de registros inconsistentes derivados de la ansiedad de los usuarios al hacer clic repetido. |
| **DRV-05** | Tolerancia a la Indisponibilidad del Sistema Legado | C05, Escenario A | Disponibilidad / Resiliencia | **Crítica (Alta / Alto)** | PIA continúa operando de forma autónoma durante caídas del sistema legado de hasta 40 minutos sin bloquear inscripciones válidas ni detener consultas. | El sistema legado es un punto único de falla (SPOF) externo. Desacoplarlo es la única manera de evitar que sus fallas paralicen el período de matrícula. |
| **DRV-06** | Disponibilidad Operativa Mensual | C03 | Disponibilidad | **Alta (Alta / Medio)** | Uptime $\ge 99,9\%$ durante el mes de matrícula (máximo 43,8 minutos de indisponibilidad no programada). | La ventana de matrícula es de misión crítica. Si el sistema falla, se retrasan los calendarios académicos y se desbordan las oficinas de registro. |
| **DRV-07** | Recuperación ante Desastres (RTO y RPO) | C06, C07 | Resiliencia / Continuidad | **Alta (Alta / Alto)** | $\text{RTO} \le 30 \text{ min}$, $\text{RPO} \le 5 \text{ min}$ ante falla total del nodo principal de base de datos o aplicación. | Cumple el estándar institucional de continuidad. Un RPO de 5 minutos limita la pérdida potencial a una ventana marginal de solicitudes. |
| **DRV-08** | Restricción Tecnológica On-Premise y Presupuesto | C08 | Restricción Tecnológica | **Crítica (Alta / Medio)** | Despliegue 100% sobre Linux y PostgreSQL. Cero servicios comerciales externos o licencias de nube pública añadidas. | Mandato financiero e institucional de la universidad. La arquitectura debe maximizar el rendimiento del hardware local disponible. |
| **DRV-09** | Trazabilidad, Auditoría y Seguridad de Datos | C09, Ley 1581 | Seguridad / Conformidad | **Media-Alta (Alta / Medio)** | 100% de operaciones de inscripción, cancelación y error registradas con log estructurado (IP, timestamp ISO 8601, ID usuario, hash transaccional) y RBAC activo. | Requisito legal y de auditoría interna para resolver reclamos de estudiantes sobre asignaturas borradas o cupos asignados. |
| **DRV-10** | Mantenibilidad con Equipo Reducido | C10 | Mantenibilidad | **Alta (Media / Alto)** | Arquitectura comprensible y gobernable por 4 desarrolladores y 1 administrador. Nuevas reglas académicas desplegables en $\le 2$ días laborables. | Previene la creación de un sistema monolítico inmanejable o una arquitectura de microservicios desmesurada que el equipo no pueda operar con éxito. |

---

## 6. Escenarios de Calidad Verificables (Estándar SEI)

Para garantizar la verificabilidad de los drivers, se formulan los escenarios bajo el estándar del *Software Engineering Institute* (SEI), especificando: Fuente del estímulo, Estímulo, Artefacto estimulado, Entorno/Contexto, Respuesta esperada y Medida de respuesta cuantificable.

---

### Escenario 1: Rendimiento y Latencia de Consulta bajo Concurrencia Pico
* **ID:** ESC-QA-01 (Asociado a C01, C02, DRV-02, DRV-03)
* **Fuente del estímulo:** Comunidad estudiantil de pregrado (1.200 a 1.800 estudiantes conectados al unísono).
* **Estímulo:** Solicitudes masivas concurrentes de consulta de oferta académica, horarios y cupos disponibles para el período actual.
* **Artefacto:** Módulo de Consulta de Catálogo Académico y Capa de Persistencia/Caché de PIA.
* **Entorno / Contexto:** Primeras 2 horas de apertura de la jornada de matrícula; alta carga en la red interna y saturación de conexiones.
* **Respuesta esperada:** El sistema despacha las consultas de catálogo sin degradación apreciable, entregando la información actualizada de asignaturas y disponibilidad de cupos en tiempo casi real.
* **Medida cuantificable:**
  * El **95% de las respuestas a consultas ($P_{95}$)** se entregan en **menos de 2,0 segundos**.
  * La tasa de errores de servidor (HTTP 5xx) es **menor al 0,01%**.
  * El consumo de CPU en los servidores de aplicación no supera el **75% sostenido**.

---

### Escenario 2: Consistencia Estricta de Cupos e Idempotencia ante Reintentos Masivos
* **ID:** ESC-QA-02 (Asociado a C01, C04, DRV-01, DRV-04, Escenario B)
* **Fuente del estímulo:** 200 estudiantes compitiendo simultáneamente por los últimos 5 cupos de una asignatura de alta demanda, enviando hasta 5 reintentos automáticos por lentitud en su conexión de red.
* **Estímulo:** Múltiples peticiones HTTP POST de inscripción concurrentes y redundantes dirigidas al mismo grupo académico.
* **Artefacto:** Módulo de Inscripción Transaccional y Mecanismo de Control de Concurrencia en PostgreSQL.
* **Entorno / Contexto:** Momento pico de matrícula con incremento de tráfico del 50% (1.800 concurrentes); clientes con latencia de red variable reintentando agresivamente.
* **Respuesta esperada:** 
  1. El sistema aplica un control estricto de concurrencia que adjudica exactamente los 5 cupos disponibles en orden secuencial válido.
  2. El sistema detecta las solicitudes duplicadas de un mismo usuario mediante un token idempotente y las trata como una única intención transaccional.
  3. Los estudiantes sin cupo reciben una notificación clara de "Cupo no disponible" sin que la base de datos entre en bloqueo mutuo (*deadlock*).
* **Medida cuantificable:**
  * **Cero sobrecupos ($\Delta \text{Cupos} = 0$):** El total de inscritos nunca supera el cupo máximo oficial configurado.
  * **Cero registros duplicados:** Ningún estudiante queda registrado más de una vez en el mismo grupo.
  * **Tiempo de resolución:** El 100% de los reintentos duplicados se resuelve en $\le 1,5$ segundos devolviendo el estado real de la primera transacción.

---

### Escenario 3: Disponibilidad y Resiliencia ante Indisponibilidad del Sistema Legado
* **ID:** ESC-QA-03 (Asociado a C03, C05, DRV-05, DRV-06, Escenario A)
* **Fuente del estímulo:** Falla de infraestructura en el servidor del sistema académico legado que lo deja inoperativo durante 40 minutos al inicio de la jornada de matrícula.
* **Estímulo:** Interrupción absoluta de comunicaciones vía red/base de datos con el sistema legado.
* **Artefacto:** Módulo de Integración/Sincronización y Motor de Reglas Académicas Locales de PIA.
* **Entorno / Contexto:** Proceso de matrícula en curso; miles de estudiantes intentando inscribir asignaturas.
* **Respuesta esperada:**
  1. El PIA detecta la caída del legado mediante un mecanismo de *Circuit Breaker* (interruptor automático) para no agotar hilos de ejecución en esperas infructuosas (*timeouts*).
  2. Las consultas y la validación de prerrequisitos continúan operando localmente utilizando la réplica/caché local precargada con el historial oficial de los estudiantes.
  3. Las inscripciones exitosas se registran en PIA y se encolan de forma segura en un repositorio transaccional local (*Outbox Pattern*).
  4. Una vez restablecido el legado a los 40 minutos, un proceso en segundo plano sincroniza progresivamente las inscripciones en orden cronológico sin intervención manual.
* **Medida cuantificable:**
  * **Disponibilidad de PIA para el estudiante:** **100%** de continuidad en consultas e inscripciones locales durante los 40 minutos de caída externa.
  * **Pérdida de inscripciones:** **0 transacciones perdidas**.
  * **Tiempo de reconciliación post-recuperación:** El 100% de los registros encolados se sincroniza con el sistema legado en un tiempo inferior a **15 minutos** tras el restablecimiento del servicio legado.

---

### Escenario 4: Recuperación ante Desastres Físicos o de Base de Datos (RTO / RPO)
* **ID:** ESC-QA-04 (Asociado a C06, C07, DRV-07)
* **Fuente del estímulo:** Falla de hardware catastrófica (corrupción de disco o corte abrupto de energía) en el nodo primario de base de datos PostgreSQL.
* **Estímulo:** Caída imprevista del servidor de base de datos durante una jornada activa de inscripciones.
* **Artefacto:** Mecanismo de Replicación de PostgreSQL y Procedimiento de Conmutación por Falla (*Failover*).
* **Entorno / Contexto:** Período de matrícula con transacciones vivas en memoria y en disco.
* **Respuesta esperada:** El administrador de infraestructura o un monitor de alta disponibilidad (*pg_auto_failover* o réplica de lectura promovida) detecta la caída, promueve la réplica secundaria como nodo primario de escritura y reconfigura los pools de conexiones de la aplicación.
* **Medida cuantificable:**
  * **RPO alcanzado:** Pérdida de datos transaccionales inferior a **1 minuto** (mediante *streaming replication* sincrónica o semi-sincrónica), cumpliendo con holgura el límite máximo de 5 minutos (C07).
  * **RTO alcanzado:** Sistema completamente restablecido para los usuarios en **menos de 15 minutos**, cumpliendo con holgura la meta de 30 minutos (C06).

---

### Escenario 5: Trazabilidad, Auditoría y Seguridad de Operaciones
* **ID:** ESC-QA-05 (Asociado a C09, DRV-09)
* **Fuente del estímulo:** Reclamo formal de un estudiante alegando que su cupo fue cancelado indebidamente o que el sistema no guardó su selección.
* **Estímulo:** Solicitud de auditoría técnica por parte de la secretaría académica de la facultad.
* **Artefacto:** Módulo de Auditoría y Trazabilidad Transaccional (*Audit Log Engine*).
* **Entorno / Contexto:** Post-cierre de matrículas; revisión forense de datos.
* **Respuesta esperada:** El sistema genera un informe cronológico inmutable con todas las interacciones realizadas por el usuario y los procesos del sistema sobre su matrícula, evidenciando dirección IP, agente de usuario (*User-Agent*), marca de tiempo precisa con zona horaria, acción realizada y firma/hash de integridad de la operación.
* **Medida cuantificable:**
  * **Cobertura de auditoría:** **100% de las mutaciones** (creación, edición, cancelación de inscripciones) registradas.
  * **Tiempo de generación de reporte forense:** Consulta de trazabilidad completa por estudiante ejecutable en **menos de 5 segundos**.
  * **Inmutabilidad:** Cero registros de auditoría modificables por usuarios de la aplicación o desarrolladores (almacenamiento en tabla particionada de sólo anexión con permisos restringidos).

---

### Escenario 6: Mantenibilidad y Adaptación de Reglas con Capacidad Humana Limitada
* **ID:** ESC-QA-06 (Asociado a C10, DRV-10)
* **Fuente del estímulo:** Consejo Académico de la Universidad aprueba una excepción extraordinaria: permitir a estudiantes de último semestre inscribir una materia con solapamiento de horario de hasta 1 hora o cursar un crédito extra sin prerrequisito estricto.
* **Estímulo:** Solicitud de cambio en la regla de validación de inscripciones a 3 días del inicio de matrícula.
* **Artefacto:** Motor de Reglas Académicas desacoplado y Capa de Configuración.
* **Entorno / Contexto:** Equipo técnico disponible: 4 desarrolladores y 1 administrador de infraestructura.
* **Respuesta esperada:** Dos desarrolladores parametrizan e implementan la nueva lógica en el motor de reglas independiente, ejecutan la suite de pruebas unitarias automatizadas y despliegan el cambio mediante un procedimiento seguro sin alterar el esquema de base de datos ni tocar el sistema legado.
* **Medida cuantificable:**
  * **Tiempo total de ciclo (desarrollo, pruebas y despliegue):** Inferior a **16 horas laborables** (2 días hábiles).
  * **Regresiones:** Cero afectaciones colaterales en las demás reglas académicas existentes, verificado mediante una cobertura de pruebas automatizadas superior al 85% sobre el catálogo de reglas.

---

## 7. Análisis de Trade-offs y Conflictos entre Drivers

En sistemas de alta concurrencia y criticidad transaccional, los atributos de calidad entran naturalmente en conflicto. Un buen diseño arquitectónico no ignora estas tensiones, sino que las transparenta y define compromisos (*trade-offs*) equilibrados y técnicamente viables.

```
+--------------------------------------------------------------------------------------------------+
|                                    TENSIONES ARQUITECTÓNICAS                                     |
+--------------------------------------------------------------------------------------------------+
|                                                                                                  |
|   [DRV-01: Cero Sobrecupo]  <==== CONFLICTO 1 ====>  [DRV-03: Concurrencia Masiva / Rendimiento] |
|   (Bloqueo estricto / Serialización)                  (Baja latencia / Alto Throughput)          |
|                                                                                                  |
|   [DRV-06: Disponibilidad 99.9%] <== CONFLICTO 2 ==> [DRV-05: Dependencia Sistema Legado Fallido]|
|   (Uptime continuo en PIA)                            (Caídas externas de hasta 40 min)          |
|                                                                                                  |
|   [DRV-07: Resiliencia RTO/RPO] <== CONFLICTO 3 ==>  [DRV-08: Restricción On-Premise / C10]     |
|   (Alta disponibilidad / Failover)                    (Linux/PostgreSQL, 4 devs + 1 sysadmin)    |
|                                                                                                  |
+--------------------------------------------------------------------------------------------------+
```

---

### Conflicto 1: Consistencia Estricta de Cupos (DRV-01 / C04) vs. Rendimiento y Concurrencia Masiva (DRV-02, DRV-03 / C01, C02, Escenario B)

* **Naturaleza del conflicto:**
  Garantizar que no exista un solo sobrecupo en situaciones donde cientos de estudiantes compiten simultáneamente por las últimas plazas de un grupo académico exige típicamente mecanismos de control de concurrencia pesados (como bloqueos pesimistas `SELECT ... FOR UPDATE` o transacciones a nivel de aislamiento `SERIALIZABLE`). Sin embargo, bloquear filas o tablas en PostgreSQL bajo una concurrencia de 1.200 a 1.800 usuarios provoca una severa contención en base de datos, colas de espera, bloqueos mutuos (*deadlocks*) y agotamiento del pool de conexiones, disparando la latencia de respuesta muy por encima de los 2 segundos exigidos por C02 y degradando el rendimiento general.

* **Estrategia arquitectónica de resolución propuesta:**
  1. **Separación de Responsabilidades de Consulta y Comando (Patrón CQRS Ligero):**
     * Las operaciones de **lectura masiva** (consultar catálogo, asignaturas y cupos estimados) se dirigen a una réplica de lectura en PostgreSQL o a un almacenamiento en caché en memoria local (ej. Redis open-source en Linux o caché de aplicación sincronizada), eliminando el 90% de la carga de lectura del nodo transaccional.
  2. **Reserva Atómica con Actualización Condicional (Bloqueo Optimista a Nivel de Fila):**
     * En lugar de transacciones largas y bloqueos de tabla, la reserva de cupo se ejecuta mediante una única sentencia SQL atómica que decrementa el cupo condicionalmente:
       ```sql
       UPDATE grupo_academico 
       SET cupos_disponibles = cupos_disponibles - 1,
           version = version + 1
       WHERE id_grupo = :idGrupo 
         AND cupos_disponibles > 0;
       ```
     * Si la sentencia retorna que se actualizó 0 filas, el sistema sabe de inmediato y de manera atómica que el cupo se agotó, sin haber bloqueado a los demás usuarios.
  3. **Filtro de Idempotencia en la Capa de Entrada:**
     * Para mitigar el Escenario B (reintentos repetidos hasta 5 veces), la capa API implementa un mecanismo de deduplicación con clave de idempotencia (`Idempotency-Key` generada a partir de `ID_Estudiante + Periodo + ID_Grupo`). Si una petición idéntica llega mientras la primera se está procesando o ya fue resuelta, el sistema no ejecuta una nueva transacción; simplemente espera o devuelve el resultado en caché de la primera petición, evitando el desgaste de la base de datos.

---

### Conflicto 2: Disponibilidad Continua de la PIA (DRV-06 / C03) vs. Acoplamiento y Dependencia con el Sistema Legado Inestable (DRV-05 / C05, Escenario A)

* **Naturaleza del conflicto:**
  La universidad exige una disponibilidad mensual del 99,9% durante el período de matrícula (máximo 43,8 minutos de caída al mes). Sin embargo, la condición C05 y el Escenario A establecen que el sistema académico legado puede dejar de responder durante **hasta 40 minutos**. Si la nueva plataforma PIA consulta los prerrequisitos o sincroniza las inscripciones de forma síncrona (bloqueante) con el legado en cada solicitud, cualquier fallo o lentitud del sistema legado arrastrará inevitablemente a la PIA hacia la indisponibilidad total, incumpliendo de inmediato la meta de disponibilidad C03 y deteniendo el proceso de matrícula.

* **Estrategia arquitectónica de resolución propuesta:**
  1. **Patrón Snapshot / Read-Replica de Prerrequisitos e Historial (Desacoplamiento de Lectura):**
     * Antes de abrir la ventana de matrícula, la PIA ejecuta un proceso de extracción e ingesta que consolida una copia local optimizada de sólo lectura en PostgreSQL con: la malla curricular, las reglas de prerrequisitos vigentes y el historial académico cerrado de los 24.000 estudiantes hasta el período inmediatamente anterior.
     * La validación de prerrequisitos durante la matrícula se realiza **100% de manera autónoma en la PIA**, sin tocar el sistema legado en tiempo real.
  2. **Patrón Transactional Outbox + Sincronización Asíncrona (Desacoplamiento de Escritura):**
     * Cuando un estudiante inscribe una asignatura, la PIA confirma la transacción localmente de inmediato (reserva el cupo, registra la inscripción con estado `PENDIENTE_SINCRONIZACION` y emite el comprobante de pre-matrícula al estudiante).
     * En la misma transacción de base de datos local, se escribe un evento en una tabla `outbox_inscripciones`.
     * Un servicio de fondo (*worker*) en Linux toma los eventos de la bandeja de salida y los envía de forma asíncrona al sistema legado mediante llamadas controladas con reintentos exponenciales y *circuit breaker*.
  3. **Manejo de Caídas de 40 Minutos (Escenario A):**
     * Si el sistema legado se cae, el *Circuit Breaker* se abre; las inscripciones continúan registrándose con normalidad en la PIA y se acumulan de manera segura en el Outbox local.
     * Al estudiante se le informa con transparencia pedagógica: *"Tu inscripción ha sido confirmada con éxito por la PIA con reserva de cupo garantizada; la sincronización con el historial oficial consolidado se completará automáticamente en breve"*.
     * Al restablecerse el legado, el *worker* retoma el vaciado de la cola en orden estricto (FIFO), sin saturar el sistema antiguo.

---

### Conflicto 3: Alta Continuidad y Recuperación Rápida (RTO $\le$ 30 min, RPO $\le$ 5 min / C06, C07) vs. Restricción Tecnológica On-Premise y Equipo Reducido (C08, C10)

* **Naturaleza del conflicto:**
  Alcanzar un RTO de 30 minutos y un RPO de 5 minutos usualmente se soluciona en la industria contratando infraestructura en la nube administrada con *multi-region failover*, balanceadores automáticos y servicios administrados costosos. Sin embargo, la universidad restringe el despliegue a **infraestructura local propia (Linux y PostgreSQL)**, sin servicios comerciales adicionales (C08), y con un equipo de operaciones de **un único administrador de infraestructura y cuatro desarrolladores** (C10). Implementar soluciones de alta disponibilidad extremadamente complejas (como clusters multi-master distribuidos o arquitecturas de microservicios con orquestadores Kubernetes a gran escala) desbordaría la capacidad operativa del equipo, provocando errores humanos y caídas operativas más prolongadas.

* **Estrategia arquitectónica de resolución propuesta:**
  1. **Arquitectura Modular Monolítica Desacoplada (Pragmática y Sostenible):**
     * En lugar de fragmentar el sistema en 10 microservicios independientes que demanden mallas de servicios (*service mesh*) y un esfuerzo desmedido de monitoreo, la PIA se concibe como una arquitectura modularizada y limpia desplegada en contenedores ligeros Linux (o servicios `systemd`), fácil de depurar, versionar y mantener por los 4 desarrolladores.
  2. **Replicación Nativa de PostgreSQL en Modo Streaming (Physical Streaming Replication):**
     * Se configura un nodo maestro (Primario) y un nodo esclavo (Standby/Réplica) en dos servidores físicos Linux independientes de la universidad.
     * La replicación nativa por streaming envía continuamente los registros WAL (*Write-Ahead Logging*), asegurando que la réplica esté al día con una diferencia de milisegundos respecto al nodo primario, garantizando un **RPO $< 1$ minuto** (superando holgadamente la meta de 5 minutos de C07).
  3. **Mecanismo de Conmutación Simplificado (*Failover* Semi-Automatizado):**
     * Se implementa una herramienta open-source estándar y madura en el ecosistema Linux/PostgreSQL como **`pg_auto_failover`** o un balanceador de carga interno **HAProxy/Keepalived** con *Virtual IP (VIP)*.
     * Ante una falla irrecuperable del primario, la promoción de la réplica a primario toma entre 2 y 5 minutos, permitiendo que el RTO real sea inferior a 15 minutos, muy por debajo de los 30 minutos máximos permitidos por C06, y totalmente operable por el administrador institucional sin sobrecarga de complejidad.

---

## 8. Criterios de Orientación para las Siguientes Fases de Diseño

Los drivers identificados y priorizados en este documento establecen las directrices ineludibles para los diagramas de componentes, despliegue y modelos de datos de las fases posteriores:

1. **Prioridad número uno en la capa de datos:** El modelo relacional en PostgreSQL debe estructurarse para soportar concurrencia a nivel de fila y aislamiento estricto de cupos, evitando cualquier bloqueo que paralice lecturas.
2. **Autonomía de la lógica de negocio:** El motor de prerrequisitos de la PIA debe ser una librería interna desacoplada, ejecutable contra datos locales pre-consolidados.
3. **Manejo explícito de la asincronía:** El flujo de sincronización con el sistema legado debe ser diseñado desde el día uno como asíncrono y tolerante a fallas mediante el patrón Outbox.
4. **Idempotencia end-to-end:** Todo endpoint HTTP que mute el estado de una inscripción debe requerir obligatoriamente un token de idempotencia transmitido desde el cliente web.
5. **Simplicidad arquitectónica:** Cada componente tecnológico propuesto en los siguientes entregables debe justificar cómo puede ser operado con soltura por los 4 desarrolladores y el administrador de infraestructura disponibles.
