# Propuesta de Arquitectura: Plataforma de Inscripción Académica (PIA)
## Universidad de la Salle — Evaluación Parcial 2

* **Estudiante:** Noah Van Otterdijk Silva  
* **Documento de Identidad:** 1023030297  
* **Usuario / Correo Institucional:** `notterdijk97`  
* **Repositorio Oficial:** `parcial2_notterdijk97`  
* **Rama de Trabajo:** `dev_notterdijk97`  
* **Escenario Extraordinario Asignado:** **Escenario B (Último dígito impar = 7)**  
  * *Condición:* Aumento del 50% en usuarios concurrentes (de 1.200 a 1.800 estudiantes) y hasta 5 solicitudes redundantes por reintentos compulsivos.

---

## 1. Resumen Ejecutivo de la Propuesta

La Universidad de la Salle cuenta con una comunidad de aproximadamente **24.000 estudiantes**, **1.100 docentes** y **cinco facultades**. Durante más de 12 años, las inscripciones de asignaturas se han gestionado a través de un sistema académico monolítico fuertemente acoplado que colapsa sistemáticamente en las primeras horas de matrícula. Los síntomas principales incluyen lentitud extrema, inscripciones duplicadas, inconsistencias de sobrecupo y una dependencia síncrona fatal con un sistema legado que sufre caídas de hasta 40 minutos.

Como arquitecto de software del proyecto, la propuesta para la nueva **Plataforma de Inscripción Académica (PIA)** se fundamenta en un diseño pragmático, altamente resiliente y estrictamente ajustado a las restricciones de la universidad: **infraestructura local propia Linux y PostgreSQL sin costos de licencias comerciales (C08)** y operada de forma sostenible por un equipo de **cuatro desarrolladores y un administrador de sistemas (C10)**.

### Pilares Fundamentales de la Solución:

1. **Estilo Arquitectónico (Monolito Modular Hexagonal):**  
   Rechazamos la sobreingeniería de microservicios distribuidos que desbordaría a un equipo de 5 personas. Adoptamos un **Monolito Modular con Arquitectura Limpia/Hexagonal**, garantizando límites de dominio estrictos en código, llamadas en memoria a velocidad de CPU (nanosegundos) y transacciones relacionales ACID locales sin la complejidad ni el riesgo de sobrecupo de transacciones distribuidas.
2. **Mitigación Frontal del Escenario B (Concurrencia +50% y Reintentos):**  
   Ante 1.800 usuarios concurrentes y ráfagas de hasta 5 clics automáticos por estudiante, implementamos una estrategia de defensa en cuatro capas:
   * **Capa 1 (Frontend):** Deshabilitación visual del botón de envío tras el primer clic y generación de cabecera `Idempotency-Key` (UUIDv4).
   * **Capa 2 (Perímetro):** Directivas `limit_req` en **Nginx** para amortiguar ráfagas de red.
   * **Capa 3 (Aplicación/Memoria):** Interceptor `IdempotencyGuard` apoyado en **Redis** (`SET NX EX 60s`) que absorbe los 4 reintentos duplicados en $< 2$ ms sin tocar la base de datos.
   * **Capa 4 (Base de Datos):** Restricción declarativa `UNIQUE (id_estudiante, id_grupo, periodo)` en PostgreSQL.
3. **Garantía Absoluta de Cero Sobrecupo ($\Delta \text{Cupo} = 0$, C04):**  
   Eliminamos los bloqueos pesimistas (`SELECT FOR UPDATE`) que provocan *deadlocks* y colas de espera masivas. La reserva se realiza mediante una única sentencia SQL atómica condicional:
   ```sql
   UPDATE grupo_academico 
   SET cupos_disponibles = cupos_disponibles - 1 
   WHERE id_grupo = :id AND cupos_disponibles > 0;
   ```
   Aprovechando el motor MVCC de PostgreSQL, la cerradura de tupla dura microsegundos. Si retorna 0 filas afectadas, se ejecuta un *Rollback* inmediato y se notifica al estudiante sin encolar a los demás.
4. **Desacoplamiento Total del Sistema Legado (Escenario A y C05):**  
   Para blindar a la PIA de las caídas de 40 minutos del sistema viejo y cumplir la **disponibilidad mensual del 99,9% (C03)**:
   * **Lecturas:** Se precarga un *Snapshot local* con el historial oficial y la malla curricular, permitiendo validar prerrequisitos de forma 100% autónoma en la PIA.
   * **Escrituras:** Se implementa el patrón **Transactional Outbox**. La confirmación al estudiante es inmediata a nivel local. Un worker en Linux procesa la cola asíncronamente con un **Circuit Breaker** que suspende llamadas cuando el legado cae y drena los eventos automáticamente al restablecerse, con cero pérdida de datos (RPO $\le 5$ min, C07).

---

## 2. Pila Tecnológica Seleccionada (100% On-Premise y Open Source)

En estricto cumplimiento de la restricción **C08** (sin costos comerciales adicionales) y **C10** (equipo acotado):

| Capa / Componente | Tecnología Seleccionada | Justificación Técnica en la Arquitectura |
| :--- | :--- | :--- |
| **Proxy & Balanceo** | **Nginx** (Linux) | Terminación TLS 1.3, compresión gzip/brotli, micro-caché y *Rate Limiting* perimetral ante reintentos. |
| **Cliente Web** | **SPA en TypeScript / React** | Interfaz reactiva, validación de formularios en cliente, gestión de tokens JWT y generación de `Idempotency-Key`. |
| **Backend Central** | **Monolito Modular Hexagonal** (Node.js / Go / Spring Boot) | API REST de alto rendimiento. Módulos de Catálogo, Inscripción, Reglas Académicas y Auditoría desacoplados. |
| **Caché & Idempotencia** | **Redis Open-Source** (Linux) | Caché de oferta académica ($P_{95} \le 2$ s) y control atómico de deduplicación de reintentos (`SET NX`). |
| **Base de Datos** | **PostgreSQL 16** (Linux) | Motor transaccional ACID primario para reservas de cupos, persistencia de matrículas y tabla Outbox. |
| **Pooler de Conexiones** | **PgBouncer** (Linux) | Agrupador de conexiones (*Transaction Pooling*); permite a 1.800 concurrentes compartir 60-80 conexiones reales a PostgreSQL. |
| **Alta Disponibilidad** | **Streaming Replication WAL** | Replicación física continua de PostgreSQL hacia un nodo Standby local (RPO $< 1$ min, RTO $< 15$ min, C06/C07). |
| **Pruebas y Validación** | **k6** + **dependency-cruiser** / **ArchUnit** | Pruebas de estrés automatizadas y verificación de reglas de arquitectura como código en el pipeline de CI. |

---

## 3. Estructura del Repositorio y Guía de Navegación

Toda la documentación técnica y los diagramas arquitectónicos se encuentran organizados dentro de la carpeta `/docs`:

```
parcial2_notterdijk97/
├── README.md                                    # Resumen ejecutivo y guía de navegación (este archivo)
└── docs/
    ├── actividad1.md                            # Actividad 1: Drivers, Escenarios SEI y Trade-offs
    ├── actividad2.md                            # Actividad 2: Estructura, Justificación y Modelo C4
    ├── actividad3/                              # Actividad 3: Registros de Decisiones Arquitectónicas (ADR)
    │   ├── adr-001-seleccion-estilo-arquitectonico.md
    │   ├── adr-002-consistencia-cupos-inscripciones.md   (Mitigación Escenario B)
    │   └── adr-003-integracion-fallas-sistema-legado.md  (Tolerancia a caídas Escenario A)
    ├── actividad4.md                            # Actividad 4: 4 Tácticas Arquitectónicas con Validación k6
    ├── actividad5.md                            # Actividad 5: Modelo de Gobernanza, CI/CD y PR Gatekeeping
    └── diagramas/                               # Diagramas Vectoriales C4 (SVG)
        ├── c4_nivel1_contexto.svg               # C4 Nivel 1: Contexto Institucional
        ├── c4_nivel2_contenedores.svg           # C4 Nivel 2: Contenedores en Linux
        └── c4_nivel3_componentes.svg            # C4 Nivel 3: Descomposición de Cupos e Inscripciones
```

### Enlaces Directos a los Entregables:

* 📄 **[Actividad 1: Drivers Arquitectónicos y Escenarios de Calidad](docs/actividad1.md)**  
  *Identificación de objetivos de negocio, restricciones, atributos de calidad C01-C10, matriz de priorización, 6 escenarios en estándar SEI y análisis de trade-offs.*
* 📄 **[Actividad 2: Estructura Arquitectónica y Modelo C4](docs/actividad2.md)**  
  *Justificación del Monolito Modular Hexagonal, diagramas C4 (Contexto, Contenedores y Componentes), límites transaccionales ACID y mecanismos de comunicación.*
* 📁 **[Actividad 3: Registros de Decisiones Arquitectónicas (ADRs)](docs/actividad3/)**  
  * **[ADR-001: Selección del Estilo Arquitectónico](docs/actividad3/adr-001-seleccion-estilo-arquitectonico.md):** Justificación de Monolito Modular frente a Microservicios.
  * **[ADR-002: Consistencia de Cupos e Idempotencia](docs/actividad3/adr-002-consistencia-cupos-inscripciones.md):** Reserva condicional y Redis frente a bloqueos pesimistas (Escenario B).
  * **[ADR-003: Integración y Manejo de Fallas del Legado](docs/actividad3/adr-003-integracion-fallas-sistema-legado.md):** Patrón Transactional Outbox y Circuit Breaker frente a llamadas síncronas.
* 📄 **[Actividad 4: Selección y Validación de Tácticas Arquitectónicas](docs/actividad4.md)**  
  *Detalle técnico de 4 tácticas (Idempotencia, Reserva Atómica, Outbox con Circuit Breaker, CQRS con réplica y caché), flujos de ejecución, comportamiento MVCC de PostgreSQL y protocolos de prueba con k6.*
* 📄 **[Actividad 5: Gobernanza de Arquitectura](docs/actividad5.md)**  
  *Políticas verificables, matriz RACI para 4 desarrolladores y 1 sysadmin, ciclo de vida de ADRs, automatización CI/CD con linter de dependencias, gestión de deuda técnica y checklist estricto de Pull Requests.*
* 🖼️ **[Carpeta de Diagramas Vectoriales C4](docs/diagramas/)**  
  *Diagramas SVG de alta fidelidad para Nivel 1 (Contexto), Nivel 2 (Contenedores) y Nivel 3 (Componentes).*

---

## 4. Matriz de Cumplimiento de Condiciones del Caso

| Código | Condición Operativa del Caso | Mecanismo de Respuesta en la Arquitectura | Documento de Respaldo |
| :---: | :--- | :--- | :--- |
| **C01** | 1.200 usuarios concurrentes | Monolito Modular, PgBouncer y separación de lecturas con Redis. | [Actividad 2](docs/actividad2.md) / [Actividad 4](docs/actividad4.md) |
| **C02** | 95% consultas $\le 2$ s | Caché de catálogo en Redis ($< 15$ ms) y réplica de lectura PostgreSQL. | [Actividad 1](docs/actividad1.md) / [Actividad 4](docs/actividad4.md) |
| **C03** | Uptime mensual $\ge 99,9\%$ | Desacoplamiento total del sistema legado vía Transactional Outbox. | [Actividad 3 (ADR-003)](docs/actividad3/adr-003-integracion-fallas-sistema-legado.md) |
| **C04** | Cero sobrecupo autorizado | `UPDATE ... WHERE cupos > 0` con Tuple Lock atómico en PostgreSQL. | [Actividad 3 (ADR-002)](docs/actividad3/adr-002-consistencia-cupos-inscripciones.md) |
| **C05** | Legado cae hasta 40 min | Snapshot local precargado y Circuit Breaker en worker desacoplado. | [Actividad 4](docs/actividad4.md) / [ADR-003](docs/actividad3/adr-003-integracion-fallas-sistema-legado.md) |
| **C06** | RTO $\le 30$ minutos | Conmutación por falla semi-automatizada con `pg_auto_failover` ($\le 15$ min). | [Actividad 1](docs/actividad1.md) / [Actividad 2](docs/actividad2.md) |
| **C07** | RPO $\le 5$ minutos | Replicación nativa semi-sincrónica por streaming de registros WAL ($< 1$ min). | [Actividad 1](docs/actividad1.md) / [Actividad 2](docs/actividad2.md) |
| **C08** | Linux y PostgreSQL On-Premise | Pila 100% de código abierto sin licencias comerciales adicionales. | [ADR-001](docs/actividad3/adr-001-seleccion-estilo-arquitectonico.md) |
| **C09** | Trazabilidad y Protección de Datos | `AuditLogger` con logs inmutables, IP, timestamp ISO y control RBAC (Ley 1581). | [Actividad 2](docs/actividad2.md) / [Actividad 5](docs/actividad5.md) |
| **C10** | 4 desarrolladores y 1 sysadmin | Arquitectura sin microservicios hipercomplejos, gobernada por CI y PRs. | [Actividad 5](docs/actividad5.md) / [ADR-001](docs/actividad3/adr-001-seleccion-estilo-arquitectonico.md) |
| **ESC-B** | **Concurrencia +50% y 5 Reintentos** | **Filtro de idempotencia en Redis (`SET NX EX 60s`) + Constraint `UNIQUE`.** | **[Actividad 4](docs/actividad4.md) / [ADR-002](docs/actividad3/adr-002-consistencia-cupos-inscripciones.md)** |

---

## 5. Instrucciones para la Revisión

1. Se recomienda iniciar la lectura con el resumen de drivers y trade-offs en **`docs/actividad1.md`**.
2. Continuar con la visualización de la estructura y diagramas C4 en **`docs/actividad2.md`** (o inspeccionar los archivos vectoriales en **`docs/diagramas/`**).
3. Evaluar los fundamentos técnicos y de negocio de cada decisión en los registros independientes en **`docs/actividad3/`**.
4. Revisar la implementación detallada y los protocolos de prueba cuantitativa en **`docs/actividad4.md`**.
5. Concluir con las reglas de control, pipelines automatizados y criterios de Pull Request en **`docs/actividad5.md`**.
