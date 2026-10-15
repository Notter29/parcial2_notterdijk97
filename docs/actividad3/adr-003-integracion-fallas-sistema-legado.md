# ADR-003: Estrategia de Integración Asíncrona y Tolerancia a Fallas con el Sistema Académico Legado (Escenario A y C05)

* **Estado:** Aprobado  
* **Fecha:** Octubre de 2026  
* **Autor:** Noah Van Otterdijk Silva - 1023030297  
* **Proyecto:** Plataforma de Inscripción Académica (PIA) - Universidad de la Salle  

---

## 1. Contexto y Problema

El sistema académico institucional de la Universidad de la Salle fue desarrollado hace más de 12 años y continúa siendo la fuente oficial de la historia académica previa, las notas consolidadas, los planes de estudio y las mallas de prerrequisitos. La nueva Plataforma de Inscripción Académica (PIA) tiene la responsabilidad de administrar las inscripciones del período en curso y sincronizarlas de manera fidedigna hacia dicho sistema legado.

Sin embargo, el sistema legado presenta una debilidad operativa severa: **sufre interrupciones no programadas de hasta 40 minutos (Condición C05 y Escenario A)**. 

Al mismo tiempo, la universidad impone a la nueva PIA una **disponibilidad mensual no inferior al 99,9% durante los períodos de matrícula (Condición C03)**. En términos matemáticos, un 99,9% de disponibilidad en un mes de 30 días permite un tiempo máximo acumulado de indisponibilidad no planificada de apenas **43,8 minutos**. Si la PIA mantiene una dependencia síncrona o bloqueante con el sistema legado, una única caída de 40 minutos del sistema viejo consumiría de golpe el 91% del margen de indisponibilidad mensual de la nueva plataforma, paralizando el proceso de matrícula de los 24.000 estudiantes.

El problema a resolver es: **¿Cómo estructurar la integración entre ambos sistemas para que la PIA opere de forma continua y autónoma durante las caídas de 40 minutos del sistema legado, garantizando que no se generen inscripciones inválidas, confirmaciones incorrectas ni transacciones perdidas?**

---

## 2. Drivers Arquitectónicos Relacionados

* **DRV-05 (Tolerancia a la Indisponibilidad del Sistema Legado):** La PIA debe continuar operando de forma autónoma durante interrupciones del legado de hasta 40 minutos sin bloquear inscripciones válidas ni detener consultas (Condición C05 y Escenario A).
* **DRV-06 (Disponibilidad Operativa Mensual):** Mantener un uptime mensual no inferior al 99,9% durante el mes de matrícula (Condición C03).
* **DRV-07 (Recuperación y Cero Pérdida de Datos - RPO/RTO):** Asegurar que ninguna transacción confirmada localmente se pierda ante caídas externas, con RPO $\le 5$ minutos y RTO $\le 30$ minutos (Condiciones C06 y C07).
* **DRV-09 (Trazabilidad y Control de Estados):** Identificar con total claridad el estado de sincronización de cada inscripción (localmente reservada vs. confirmada en el legado oficial) para efectos de auditoría y soporte (Condición C09).

---

## 3. Alternativas Técnicamente Viables

### Alternativa 1: Sincronización Síncrona Bloqueante (RPC / HTTP Directo)
En cada solicitud de inscripción, la PIA invoca sincrónicamente los servicios del sistema legado (REST, SOAP o llamada a base de datos externa) para validar los prerrequisitos en línea y registrar de inmediato la asignatura en la base de datos antigua antes de confirmar al estudiante.

### Alternativa 2: Enlace Bidireccional Directo en Base de Datos (DBLink / Foreign Data Wrappers con Triggers)
Conectar la base de datos PostgreSQL de la PIA directamente a la base de datos relacional del sistema legado mediante extensiones como `postgres_fdw` o `dblink`. Un disparador (*trigger*) en PostgreSQL ejecuta la inserción remota en el sistema legado dentro de la misma transacción relacional del estudiante.

### Alternativa 3: Snapshot Local de Prerrequisitos + Patrón Transactional Outbox Asíncrono con Circuit Breaker (SELECCIONADA)
Desacoplar la lectura y la escritura en dos mecanismos independientes:
1. **Desacoplamiento de Lectura (Snapshot Local Precargado):** Antes de la jornada de matrícula, un proceso en lote extrae la malla curricular, las reglas de prerrequisitos vigentes y el historial académico cerrado de los 24.000 estudiantes a tablas optimizadas de sólo lectura en el PostgreSQL local de la PIA. La validación de prerrequisitos durante la matrícula se ejecuta 100% de manera autónoma en la PIA, sin tocar el sistema legado.
2. **Desacoplamiento de Escritura (Transactional Outbox):** Cuando el estudiante inscribe una materia, la PIA reserva el cupo, crea la inscripción con estado `PENDIENTE_SINCRONIZACION` e inserta un evento en la tabla local `outbox_inscripciones`, todo dentro de la misma transacción local ACID.
3. **Worker en Linux con Circuit Breaker:** Un servicio en segundo plano (*daemon*) en Linux lee los eventos de la tabla Outbox y los despacha al sistema legado. Si el legado no responde, un *Circuit Breaker* se abre de inmediato, protegiendo a la PIA de bloqueos, acumulando las inscripciones en la base de datos local y reanudando la sincronización en lotes ordenados tan pronto el legado se restablezca tras los 40 minutos.

---

## 4. Criterios de Comparación

1. **Aislamiento ante Fallas Externas:** Capacidad de la PIA para seguir operando sin degradación cuando el sistema legado cae por 40 minutos (C05 / Escenario A).
2. **Cumplimiento del Uptime del 99,9% (C03):** Independencia de la disponibilidad de la PIA respecto a la disponibilidad del sistema legado.
3. **Latencia de Respuesta al Usuario (C02):** Tiempo que tarda el estudiante en recibir la confirmación de su cupo.
4. **Gobernabilidad y Resiliencia de Datos:** Riesgo de inconsistencias, transacciones bloqueadas o registros duplicados entre ambos sistemas.

---

## 5. Matriz de Evaluación de Alternativas

| Criterio de Comparación | Alternativa 1: Sincronización Síncrona | Alternativa 2: DBLink / Triggers | Alternativa 3: Snapshot + Transactional Outbox |
| :--- | :--- | :--- | :--- |
| **Aislamiento de Fallas (C05)** | **Nulo:** Si el legado se cae 40 minutos, el 100% de las inscripciones en la PIA fallan de inmediato arrojando error al estudiante. | **Nulo:** La transacción en PostgreSQL local se cuelga esperando la respuesta del socket de base de datos remota hasta agotar el timeout. | **Excelente:** La PIA continúa registrando inscripciones y asignando cupos con total normalidad durante los 40 minutos de caída del legado. |
| **Disponibilidad 99,9% (C03)** | **Imposible de cumplir:** La PIA hereda la baja disponibilidad del sistema legado, violando el acuerdo de nivel de servicio institucional. | **Imposible de cumplir:** El pool de conexiones de la PIA colapsa con transacciones pendientes hacia la base de datos vieja caída. | **Totalmente garantizado:** El uptime de la PIA es independiente de la salud del sistema legado. |
| **Latencia ($P_{95} \le 2\text{ s}$)** | **Deficiente:** Suma la latencia del sistema legado (frecuentemente de 3 a 8 segundos en hora pico) a la transacción de la PIA. | **Deficiente:** Transacciones distribuidas a nivel de red aumentan el tiempo de retención de bloqueos. | **Excelente:** La transacción es 100% local en PostgreSQL y se confirma al estudiante en $< 150$ milisegundos. |
| **Integridad de Datos** | **Riesgo Medio:** Timeouts de red pueden causar que la materia quede registrada en el legado pero no confirmada en la PIA. | **Riesgo Alto:** Fallas parciales dejan bloqueos mutuos distribuidos difíciles de depurar por 1 sysadmin. | **Excelente:** Patrón probado de entrega garantizada (*at-least-once* con deduplicación por clave de evento). Cero transacciones perdidas. |

---

## 6. Decisión Seleccionada y Justificación Técnica

**Se decide adoptar la Alternativa 3: Snapshot Local de Prerrequisitos para Lecturas y Patrón Transactional Outbox Asíncrono con Circuit Breaker para Sincronización hacia el Sistema Legado.**

### Justificación:
1. **Ruptura del acoplamiento temporal y cumplimiento estricto de C03 (99,9% Uptime):** La única forma de aislar a la PIA de la inestabilidad de un sistema con más de 12 años es desacoplar temporalmente la solicitud del estudiante de la confirmación oficial en el legado. Al utilizar un *Snapshot local* de prerrequisitos, la PIA valida la elegibilidad del estudiante a velocidad de lectura local. Al utilizar *Transactional Outbox*, la reserva del cupo se consolida localmente de forma inmediata e irrevocable.
2. **Respuesta definitiva ante el Escenario A (Caída de 40 minutos del legado):**
   * **Comportamiento durante la contingencia:** Cuando el legado deja de responder, el *Circuit Breaker* del worker se abre tras detectar 3 fallas consecutivas o timeouts superiores a 2 segundos. A partir de ese momento, la PIA no realiza llamadas externas inútiles; las inscripciones se siguen aprobando localmente en PostgreSQL y se almacenan como eventos en la tabla `outbox_inscripciones` con estado `PENDIENTE`.
   * **Experiencia de usuario transparente:** El estudiante recibe su comprobante de inscripción inmediatamente con estado *"Confirmado en PIA - Cupo Asegurado; sincronización con historia oficial en proceso"*. No experimenta errores ni lentitud.
   * **Recuperación automática post-contingencia:** Al transcurrir los 40 minutos y restablecerse el sistema legado, el *Circuit Breaker* pasa a estado *Half-Open* y luego *Closed*. El worker en Linux comienza a procesar los eventos acumulados en lotes controlados (utilizando `SELECT ... FOR UPDATE SKIP LOCKED` para no generar contención), sincronizando el 100% de los registros en menos de 15 minutos sin intervención manual del administrador.
3. **Cero pérdida de datos (RPO = 0 para la PIA):** Dado que el evento de sincronización se inserta en la misma transacción relacional de base de datos que asigna el cupo, no existe ningún escenario donde una inscripción sea aprobada en la PIA y el evento se pierda por una falla de energía o caída de red.

---

## 7. Consecuencias y Riesgos

### Consecuencias Positivas:
* **Alta resiliencia institucional:** El proceso de matrícula de la Universidad de la Salle no vuelve a depender de la estabilidad del sistema legado.
* **Trazabilidad clara de estados:** La secretaría académica y los coordinadores pueden consultar en tiempo real qué asignaturas ya fueron impactadas en el sistema legado y cuáles están en cola de sincronización.
* **Operación sustentable para el equipo (C10):** El patrón Outbox sobre PostgreSQL no requiere servidores de mensajería complejos adicionales (como Kafka o RabbitMQ empresariales), manteniendo la arquitectura ligera y gobernable.

### Consecuencias Negativas y Desafíos:
* **Consistencia Eventual:** Existe una ventana de tiempo (habitualmente de 2 a 10 segundos en operación normal, y de hasta 40 minutos durante contingencias) en la que el sistema legado no refleja de inmediato la inscripción que el estudiante ya tiene garantizada en la PIA.
* **Gestión de fallas semánticas en el legado:** Aunque la PIA valida rigurosamente los prerrequisitos con el snapshot, existe el riesgo residual de que el sistema legado rechace una sincronización por bloqueos internos de su base de datos antigua.

### Mitigación de Riesgos:
* Implementar una **Cola de Errores / Dead Letter Queue (DLQ)** a nivel de base de datos (`outbox_fallas_revisar`) con alertas automáticas por correo electrónico para que el equipo de soporte pueda revisar cualquier rechazo excepcional.
* La política institucional definirá que la asignación de cupo en la PIA tiene validez jurídica institucional y prevalece sobre retrasos de sincronización del sistema legado.

---

## 8. Condiciones para Revisar esta Decisión

Esta decisión arquitectónica deberá revisarse si:
1. La universidad decide reemplazar y dar de baja definitivamente el sistema académico legado, migrando el repositorio oficial histórico directamente a la base de datos de la PIA, lo que eliminaría la necesidad del patrón Outbox de integración.
2. El sistema legado es modernizado con una arquitectura basada en eventos que soporte transacciones en tiempo real con latencias sub-segundo y una disponibilidad contractual del 99,99%.
