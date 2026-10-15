# Propuesta Arquitectónica: Plataforma de Inscripción Académica (PIA)
## Universidad de la Salle
**Documento de Trabajo:** Actividad 5 - Gobernanza de Arquitectura  
**Estudiante:** Noah Van Otterdijk Silva - 1023030297  
**Escenario Asignado:** Escenario B (Último dígito impar = 7: Concurrencia +50% y hasta 5 reintentos por usuario)  
**Fecha:** Octubre de 2026  
**Versión:** 1.0  

---

## 1. Enfoque y Justificación del Modelo de Gobernanza

Como estudiante de ingeniería de software a cargo de la arquitectura de la PIA, entiendo que una arquitectura no se mantiene en pie por la calidad de sus diagramas iniciales, sino por los **mecanismos de disciplina y control que evitan su degradación con el paso del tiempo**. El sistema legado de 12 años colapsó precisamente porque durante más de una década se acumularon procedimientos almacenados improvisados, reglas de negocio dispersas y accesos directos a la base de datos sin ningún marco de gobernanza.

Sin embargo, el mayor error en gobernanza es diseñar burocracias corporativas desproporcionadas que ahoguen a un equipo ágil. La gobernanza de la PIA debe ajustarse estrictamente a la realidad de la Universidad de la Salle:
* **Equipo institucional de cinco (5) personas:** Cuatro (4) desarrolladores de software y un (1) administrador de infraestructura (Condición C10).
* **Infraestructura On-Premise (Linux y PostgreSQL):** Sin presupuesto para herramientas comerciales de gobierno o suites SaaS de pago recurrente (Condición C08).
* **Criterio Clave:** Las reglas deben ser **verificables de forma automatizada en el repositorio de código** y las responsabilidades deben estar asignadas con nombres de roles concretos.

---

## 2. Políticas y Reglas Arquitectónicas Verificables

Se establecen tres reglas arquitectónicas ineludibles. Cada una responde directamente a los drivers priorizados y puede ser auditada de forma objetiva:

```
+----------------------------------------------------------------------------------------------------+
|                                REGLAS ARQUITECTÓNICAS VERIFICABLES                                 |
+----------------------------------------------------------------------------------------------------+
|                                                                                                    |
|  [POL-01: Aislamiento del Dominio]                                                                 |
|  El núcleo de reglas académicas (prerrequisitos, topes de crédito) NO puede importar paquetes      |
|  de persistencia, frameworks web ni librerías de infraestructura.                                  |
|                                                                                                    |
|  [POL-02: Transacciones Libres de Contención e Idempotencia Obligatoria]                          |
|  Toda mutación de inscripción debe exigir cabecera 'Idempotency-Key' y ejecutar reserva condicional|
|  (UPDATE ... WHERE cupos > 0). Se PROHÍBE terminantemente el uso de SELECT ... FOR UPDATE sin TTL.|
|                                                                                                    |
|  [POL-03: Desacoplamiento Estricto del Sistema Legado]                                             |
|  Se PROHÍBE cualquier llamada HTTP/RPC o conexión síncrona directa hacia el sistema legado        |
|  dentro de controladores o transacciones web. Toda sincronización debe ser vía Outbox asíncrono.   |
|                                                                                                    |
+----------------------------------------------------------------------------------------------------+
```

---

### POL-01: Aislamiento Estricto del Dominio Académico (Arquitectura Hexagonal)
* **Descripción:** El módulo de reglas de negocio (`core/domain/rules`) contiene la lógica de prerrequisitos, correquisitos, cruces de horarios y límites de créditos. Este módulo debe ser código puro sin dependencias de frameworks web (Express/Fastify/Spring Web), ni del ORM/driver de base de datos (`pg`, TypeORM, Hibernate), ni de librerías externas.
* **Driver Relacionado:** DRV-10 (Mantenibilidad en $\le 2$ días hábiles con 4 desarrolladores) y DRV-05 (Autonomía de reglas ante caídas del legado).
* **Mecanismo de Verificación:** Regla automatizada de análisis estático en el pipeline de Integración Continua (CI) mediante **ArchUnit** (o `dependency-cruiser` en TypeScript):
  * *Condición de falla:* Si cualquier archivo dentro de `src/modules/*/domain/` importa paquetes de `src/infrastructure/` o `@nestjs/*` / `express` / `pg`, la compilación se aborta con código de salida `1`.

---

### POL-02: Idempotencia Obligatoria y Reserva Atómica sin Bloqueos Pesados
* **Descripción:** Para mitigar el **Escenario B (hasta 5 clics repetidos y 1.800 concurrentes)** y garantizar **cero sobrecupo (C04)**:
  1. Todo endpoint HTTP que cree, modifique o cancele una inscripción (`POST`, `PUT`, `DELETE` en `/api/v1/inscripciones/*`) debe implementar obligatoriamente el interceptor de cabecera `Idempotency-Key`.
  2. La reserva de cupos debe realizarse exclusivamente mediante la instrucción atómica condicional:
     ```sql
     UPDATE grupo_academico SET cupos_disponibles = cupos_disponibles - 1 WHERE id = :id AND cupos_disponibles > 0;
     ```
  3. Queda **estrictamente prohibido** el uso de bloqueos pesimistas de lectura como `SELECT ... FOR UPDATE` sin un timeout explícito inferior a 500 ms, así como sentencias de bloqueo de tabla completa (`LOCK TABLE`).
* **Driver Relacionado:** DRV-01 (Cero sobrecupo), DRV-04 (Manejo de reintentos) y DRV-02 (Latencia $P_{95} \le 2$ s).
* **Mecanismo de Verificación:** 
  * Prueba de integración automatizada en CI que ejecuta dos peticiones idénticas concurrentes y valida que sólo una toque la base de datos.
  * Regla de linter de SQL / pruebas que analiza que no existan sentencias `FOR UPDATE` abiertas en los repositorios de cupos.

---

### POL-03: Prohibición de Sincronización Síncrona con Sistemas Externos
* **Descripción:** Ningún hilo de ejecución que atienda una petición HTTP proveniente de un estudiante puede abrir una conexión de red bloqueante (HTTP, SOAP, TCP o DBLink) hacia el **Sistema Académico Legado**. Toda comunicación hacia el legado debe desacoplarse mediante la inserción de un evento en la tabla local `outbox_inscripciones`.
* **Driver Relacionado:** DRV-05 (Tolerancia a caídas de 40 min del legado) y DRV-06 (Disponibilidad mensual $\ge 99,9\%$).
* **Mecanismo de Verificación:**
  * Prueba de arquitectura que escanea los controladores y servicios de aplicación (`src/modules/inscripcion/application/`): ningún servicio de aplicación puede inyectar o instanciar clientes HTTP que apunten a la URL del sistema legado.
  * Solo el contenedor desacoplado `OutboxWorker` tiene autorización de invocar el adaptador del sistema legado.

---

## 3. Roles y Responsabilidades en la Toma de Decisiones

Dado el tamaño del equipo institucional (**4 desarrolladores y 1 administrador de infraestructura**, Condición C10), los roles de gobernanza se asignan sobre las personas reales que operan la plataforma, evitando comités burocráticos ficticios:

### Estructura de Roles del Equipo PIA

| Rol Institucional | Asignación en el Equipo | Responsabilidad Principal en Gobernanza |
| :--- | :--- | :--- |
| **Líder Técnico de Arquitectura (Tech Lead)** | Desarrollador Senior (1 de los 4 devs) | Vela por la coherencia del diseño modular, aprueba los ADRs, define estándares de código y resuelve trade-offs técnicos. |
| **Desarrolladores de Software (Core Devs)** | 3 Desarrolladores | Implementan funcionalidades, crean pruebas automatizadas, proponen ADRs ante nuevos problemas y ejecutan revisiones cruzadas de código (*Peer Review*). |
| **Administrador de Infraestructura y Datos (Sysadmin/DBA)** | 1 Administrador de Sistemas | Aprueba cambios en el esquema relacional de PostgreSQL, índices, políticas de backup/replicación, dimensionamiento de hardware Linux y ajustes en Nginx/PgBouncer. |
| **Interlocutor Académico (Stakeholder)** | Director de Registro / Decanatura | Aprueba cambios funcionales en las reglas de negocio académicas (ej. modificación de prerrequisitos o fechas de matrícula). No interviene en decisiones tecnológicas. |

### Matriz de Decisión RACI (Responsible, Accountable, Consulted, Informed)

| Tipo de Decisión Arquitectónica | Tech Lead | Core Devs | Sysadmin / DBA | Interlocutor Académico |
| :--- | :---: | :---: | :---: | :---: |
| **Nuevo ADR o cambio de estilo arquitectónico** | **A** | R | C | I |
| **Cambio en el esquema relacional de PostgreSQL (Tablas, Índices, Constraints)** | R | C | **A** | I |
| **Modificación en el motor de reglas de prerrequisitos o créditos** | R | R | I | **A** |
| **Ajustes de infraestructura Linux, Nginx, Redis o PgBouncer** | C | I | **A** | I |
| **Aprobación de Pull Request a la rama `main`** | **A** | R (Peer) | C (si toca BD/Infra) | I |

*Referencias: **A** = Responsable final (*Accountable*), **R** = Ejecuta el trabajo (*Responsible*), **C** = Consultado (*Consulted*), **I** = Informado (*Informed*).*

---

## 4. Procedimiento para la Gestión del Ciclo de Vida de los ADR

Para que los Registros de Decisión Arquitectónica (ADR) no queden congelados en el tiempo, se adopta un procedimiento estándar basado en Git:

```
[1. Identificación del Problema]
                 |
                 v
[2. Redacción del ADR en rama 'adr/<slug>'] (Estado: 'Propuesto')
                 |
                 v
[3. Creación de Pull Request] (Revisión técnica de 3 días hábiles)
                 |
                 +---> Debate técnico en comentarios del PR
                 |
                 v
[4. Decisión Formal]:
       |--- Aprobado por Tech Lead (+ Sysadmin si aplica) ===> Estado: 'Aprobado' -> Merge a main
       |--- Descartado con argumentos técnicos           ===> Estado: 'Rechazado' -> Se archiva
                 |
                 v
[5. Mantenimiento Futuro]:
       Si una decisión cambia, se crea un NUEVO ADR que marca el anterior como 'Reemplazado'.
```

### Estados Permitidos de un ADR
* **Propuesto:** El documento está en redacción o bajo revisión del equipo.
* **Aprobado:** Ha sido revisado y cuenta con las firmas técnicas necesarias para su implementación.
* **Rechazado:** La alternativa no fue conveniente tras la evaluación de costos/beneficios.
* **Reemplazado:** Una decisión previa fue modificada por un nuevo ADR (se debe incluir enlace al nuevo registro: `Reemplazado por ADR-XXX`).

### Reglas del Procedimiento:
1. **Inmutabilidad de la historia:** Un ADR aprobado **nunca se edita retroactivamente** para cambiar su decisión original. Si las circunstancias cambian (ej. se contrata nube pública o cambia el motor de BD), se redacta un nuevo ADR que referencia y reemplaza al anterior.
2. **Tiempo máximo de revisión:** El equipo dispone de un máximo de **tres (3) días hábiles** para revisar y comentar una propuesta de ADR antes de emitir concepto de aprobación o rechazo.
3. **Ubicación obligatoria:** Todo ADR debe residir en el repositorio de código bajo la ruta `docs/actividad3/` (o `docs/adr/`) en formato Markdown, permitiendo que viaje versionado con el código fuente.

---

## 5. Mecanismos de Control Automático en el Repositorio (CI/CD Pipeline)

Para no depender de la memoria o la buena voluntad humana, se integran dos mecanismos automatizados en el pipeline de Integración Continua sobre servidores Linux locales (GitLab CI, Gitea Actions o GitHub Actions local):

```
                        FLUJO DE INTEGRACIÓN CONTINUA (CI)
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  [Push / Pull Request]                                                            |
|          |                                                                        |
|          v                                                                        |
|  +-----------------------------------------------------------------------------+  |
|  | MECANISMO 1: Verificación de Reglas Arquitectónicas (Linter de Arquitectura) |  |
|  | - Ejecuta: ArchUnit / dependency-cruiser en Node/Java                        |  |
|  | - Valida POL-01 (Dominio puro sin dependencias externas)                    |  |
|  | - Valida POL-03 (Servicios web no importan conector del legado)             |  |
|  | --> Si falla: ABORTA EL PIPELINE (Bloquea el PR de inmediato)               |  |
|  +-----------------------------------------------------------------------------+  |
|          | (Si pasa)                                                              |
|          v                                                                        |
|  +-----------------------------------------------------------------------------+  |
|  | MECANISMO 2: Ejecución de Pruebas Automatizadas y Cobertura                 |  |
|  | - Pruebas Unitarias de Reglas de Prerrequisitos (Cobertura >= 85%)          |  |
|  | - Pruebas de Integración con PostgreSQL en Contenedor Efímero               |  |
|  |   * Test de Reserva Atómica Condicional (Cero sobrecupo POL-02)             |  |
|  |   * Test de Deduplicación e Idempotencia (Escenario B)                      |  |
|  |   * Test del Outbox Pattern y Circuit Breaker                               |  |
|  | --> Si falla cualquier test: ABORTA EL PIPELINE                             |  |
|  +-----------------------------------------------------------------------------+  |
|          | (Si pasa)                                                              |
|          v                                                                        |
|  [PR Habilitado para Revisión Manual por Pares (Peer Review)]                     |
|                                                                                   |
+-----------------------------------------------------------------------------------+
```

---

### Mecanismo 1: Verificador Automático de Reglas Arquitectónicas (Architecture Linter)
* **Herramienta:** `dependency-cruiser` (para TypeScript/Node.js) o `ArchUnit` (para Java/Spring Boot).
* **Configuración técnica del script (`.dependency-cruiser.js`):**
  ```javascript
  module.exports = {
    forbidden: [
      {
        name: 'no-frameworks-in-domain',
        severity: 'error',
        comment: 'Violación POL-01: El núcleo de dominio no puede depender de infraestructura ni frameworks',
        from: { path: '^src/modules/[^/]+/domain' },
        to: { path: '(^src/infrastructure|^node_modules/(express|@nestjs|pg|typeorm))' }
      },
      {
        name: 'no-direct-legacy-call-from-web',
        severity: 'error',
        comment: 'Violación POL-03: Solo el OutboxWorker puede comunicarse con el sistema legado',
        from: { path: '^src/modules/inscripcion/application' },
        to: { path: '^src/infrastructure/legacy-adapter' }
      }
    ]
  };
  ```
* **Efecto de Control:** Si un desarrollador intenta importar el driver de PostgreSQL dentro de una regla de prerrequisito para "ahorrar tiempo", el pipeline falla automáticamente en menos de 10 segundos, rechazando el commit.

---

### Mecanismo 2: Suite Automatizada de Pruebas Unitarias, Integración y Regresión
* **Herramienta:** `Vitest` / `Jest` con base de datos efímera PostgreSQL mediante Docker en Linux.
* **Cobertura y Umbrales Exigidos:**
  1. **Cobertura de código mínima:** $\ge 85\%$ sobre las clases del paquete `domain/` y `application/`.
  2. **Pruebas de Integración Obligatorias:**
     * `test_cero_sobrecupo_concurrente`: Lanza 50 transacciones asíncronas sobre un grupo con 2 cupos y verifica que solo 2 pasen y 48 retornen HTTP 409.
     * `test_idempotencia_reintentos`: Envía 5 peticiones simultáneas con la misma `Idempotency-Key` y valida que la base de datos solo registre una fila en `inscripcion`.
     * `test_outbox_transactional_integrity`: Simula una falla forzada a mitad de la transacción y verifica que si la inscripción falla, el evento en `outbox_inscripciones` sufra *Rollback*.
* **Efecto de Control:** Ningún PR puede ser fusionado si la suite completa de pruebas no está 100% en verde.

---

## 6. Procedimiento para Manejo de Excepciones, Incumplimientos y Deuda Técnica

En el ciclo de vida real de un software universitario, se presentan situaciones extraordinarias (ej. una resolución rectoral de emergencia a 48 horas de abrir matrículas). La gobernanza no debe ser un obstáculo rígido, sino un marco que gestione la deuda técnica con responsabilidad:

```
[Situación Extraordinaria / Urgencia Académica]
                       |
                       v
[Solicitud de Excepción Arquitectónica (EAR - Issue en Git)]
- Justificación del negocio
- Regla arquitectónica violada (ej. POL-01 o POL-02)
- Plan de remediación y fecha de vencimiento (máx. 30 días)
                       |
                       v
[Aprobación Formal Unánime: Tech Lead + Sysadmin]
                       |
                       v
[Merge con etiqueta 'tech-debt:high-priority']
                       |
                       v
[Monitoreo y Cancelación de Deuda Técnica]:
Al vencer el plazo de 30 días, el linter de CI se reactiva automáticamente.
Si la deuda no fue pagada, el pipeline bloquea nuevos despliegues a producción.
```

### Gestión de Incumplimientos No Autorizados:
* Si un desarrollador desactiva pruebas, omite la verificación de idempotencia o burla las reglas arquitectónicas sin una EAR aprobada:
  1. El PR es rechazado de inmediato con estado **CERRADO SIN FUSIÓN**.
  2. Se abre un reporte de auditoría interna y el Tech Lead convoca una sesión de revisión técnica con el infractor para refactorizar la solución bajo los estándares establecidos.

---

## 7. Estrategia de Documentación Viva (Docs-as-Code)

Uno de los principales motivos por los cuales el sistema de hace 12 años se volvió inauditable fue que sus manuales en Word y diagramas en Visio se desactualizaron al primer mes de operaciones. 

Para la PIA se implementa la filosofía **Docs-as-Code (Documentación como Código)**:

1. **Co-ubicación con el Código Fuente:** Toda la documentación arquitectónica (Actividades 1 a 5, ADRs y diagramas C4) reside en el mismo repositorio Git dentro de la carpeta `/docs/`.
2. **Diagramas Versionables en Texto:** Todos los diagramas de arquitectura se mantienen en formato **Mermaid** y código fuente vectorial **SVG**. Esto permite que cualquier cambio en la arquitectura se visualice mediante un `git diff` textual idéntico al de cualquier clase de código.
3. **Regla de Sincronización en Pull Requests:**
   * Si un cambio de código altera un componente, agrega un contenedor o modifica el flujo de sincronización con el legado, el PR **debe incluir en el mismo commit la actualización del archivo Markdown y del diagrama correspondiente**.
   * Un PR que altere la estructura de paquetes sin actualizar los documentos en `/docs/` se considera incompleto y **no es aprobado por el Tech Lead**.

---

## 8. Criterios de Aceptación y Rechazo para Pull Requests (PR Gatekeeping)

Para unificar los criterios de revisión del equipo de 4 desarrolladores y el administrador, se establece la siguiente lista de verificación (*Checklist*) obligatoria en la plantilla de Pull Requests del repositorio:

### Lista de Verificación Obligatoria para Aprobar un PR

```markdown
### Control de Calidad del Pull Request (PIA - Universidad de la Salle)

#### 1. Verificaciones Automáticas de CI (Bloqueantes)
- [ ] El pipeline de CI ha finalizado exitosamente (100% en verde).
- [ ] El linter de arquitectura (ArchUnit / dependency-cruiser) certifica 0 violaciones a POL-01 y POL-03.
- [ ] La cobertura de pruebas automatizadas sobre el código nuevo o modificado es >= 85%.
- [ ] Las pruebas de integración de concurrencia e idempotencia pasaron sin errores.

#### 2. Revisión de Arquitectura y Rendimiento (Manual por Tech Lead)
- [ ] Si el PR agrega o muta endpoints de inscripción, ¿implementa obligatoriamente 'Idempotency-Key'?
- [ ] ¿La reserva de cupos utiliza la sentencia atómica 'UPDATE ... WHERE cupos_disponibles > 0'?
- [ ] ¿Se garantiza que NO existen llamadas síncronas bloqueantes hacia el Sistema Académico Legado?
- [ ] Si se agregaron consultas a PostgreSQL, ¿fueron revisadas con 'EXPLAIN ANALYZE' para verificar uso de índices y ausencia de escaneos secuenciales masivos (Seq Scan)?

#### 3. Revisión de Operaciones e Infraestructura (Sysadmin / DBA)
- [ ] Si hay migraciones de base de datos (`ALTER TABLE`), ¿son compatibles hacia atrás y no bloquean la tabla en caliente?
- [ ] ¿El consumo proyectado de memoria en Redis y conexiones a PgBouncer está dentro de los límites locales?

#### 4. Documentación Viva (Docs-as-Code)
- [ ] Si hubo cambios en componentes o límites transaccionales, ¿se actualizaron los diagramas C4 y documentos en `/docs/`?
- [ ] Si se adoptó un cambio estructural significativo, ¿se incluye el respectivo ADR en `/docs/actividad3/`?
```

### Regla de Rechazo Inmediato:
Cualquier Pull Request que presente al menos una de las siguientes condiciones será **rechazado y cerrado de inmediato**:
1. Fallo en cualquiera de las pruebas automatizadas del pipeline.
2. Inclusión de llamadas directas y síncronas al sistema legado dentro de controladores web.
3. Omisión de la cabecera `Idempotency-Key` en operaciones de inscripción.
4. Falta de aprobación formal por parte de al menos dos miembros del equipo (el revisor par y el Tech Lead/Sysadmin).
