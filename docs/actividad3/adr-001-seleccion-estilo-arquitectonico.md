# ADR-001: Selección del Estilo Arquitectónico para la Plataforma PIA

* **Estado:** Aprobado  
* **Fecha:** Octubre de 2026  
* **Autor:** Noah Van Otterdijk Silva - 1023030297  
* **Proyecto:** Plataforma de Inscripción Académica (PIA) - Universidad de la Salle  

---

## 1. Contexto y Problema

La Universidad de la Salle requiere renovar su plataforma de inscripciones académicas, la cual atiende una comunidad de 24.000 estudiantes y 1.100 docentes en cinco facultades. El proceso de matrícula concentra una demanda crítica en las primeras horas, con una concurrencia nominal de 1.200 usuarios simultáneos y picos extraordinarios de hasta 1.800 concurrentes (+50%, Escenario B).

Actualmente, el sistema académico legado (de más de 12 años de antigüedad) presenta colapsos recurrentes por agotamiento de conexiones y fallas de disponibilidad. La universidad debe definir la estructura y el estilo arquitectónico de la nueva solución bajo tres restricciones operativas no negociables:

1. **Infraestructura On-Premise sin costos de nube adicionales (C08):** La solución debe operar estrictamente sobre servidores Linux y motores de base de datos PostgreSQL existentes.
2. **Capacidad humana limitada para mantenimiento (C10):** El equipo a cargo está conformado únicamente por cuatro (4) desarrolladores de software y un (1) administrador de infraestructura.
3. **Alto rendimiento y baja latencia (C02):** El 95% de las consultas de catálogo debe responder en un tiempo inferior o igual a 2 segundos bajo alta carga.

El problema a resolver es: **¿Qué estilo arquitectónico garantiza la máxima resiliencia, mantenibilidad y rendimiento transaccional sin introducir una sobrecarga operativa inmanejable para el equipo institucional?**

---

## 2. Drivers Arquitectónicos Relacionados

* **DRV-10 (Mantenibilidad y Sostenibilidad con Equipo Reducido):** La solución debe ser comprensible, desplegable y operable por 4 desarrolladores y 1 administrador de sistemas (Condición C10).
* **DRV-08 (Restricción Tecnológica On-Premise):** Obligatoriedad de utilizar Linux y PostgreSQL sin contratar servicios comerciales externos o nubes públicas (Condición C08).
* **DRV-03 (Concurrencia Masiva):** Soportar entre 1.200 y 1.800 usuarios concurrentes simultáneos sin degradación del servicio (Condición C01 y Escenario B).
* **DRV-02 (Latencia de Consulta):** Percentil 95 ($P_{95}$) de latencia $\le 2,0$ segundos en consultas de catálogo y horarios (Condición C02).
* **DRV-01 (Cero Sobrecupo):** Control transaccional riguroso para impedir inscripciones por encima del cupo autorizado (Condición C04).

---

## 3. Alternativas Técnicamente Viables

### Alternativa 1: Arquitectura de Microservicios Distribuidos
Descomponer la solución en servicios autónomos desplegados en contenedores independientes (ej. Servicio de Autenticación, Servicio de Catálogo de Asignaturas, Servicio de Inscripciones y Cupos, Servicio de Auditoría y Servicio de Sincronización con el Legado), comunicados mediante HTTP/REST o gRPC con un API Gateway.

### Alternativa 2: Monolito Tradicional en Tres Capas (Layered Architecture)
Desplegar una única aplicación estructurada horizontalmente en Capa de Presentación (Controladores), Capa de Lógica de Negocio (Servicios) y Capa de Acceso a Datos (DAOs / Repositorios), compartiendo un único esquema de base de datos relacional.

### Alternativa 3: Monolito Modular con Arquitectura Limpia / Hexagonal (Modular Monolith)
Estructurar el código internamente en módulos funcionales de dominio aislados y desacoplados (Catálogo, Inscripción, Reglas Académicas, Auditoría, Integración Legada), donde cada módulo sigue el principio de Puertos y Adaptadores (Hexagonal). Se despliega como un único artefacto ejecutable en Linux (junto con un worker de fondo para el patrón Outbox) sobre PostgreSQL.

---

## 4. Criterios de Comparación

Para evaluar objetivamente las alternativas, se establecieron cuatro criterios técnicos fundamentados en la realidad del proyecto:

1. **Carga Operativa y Gobernanza (DevOps):** Esfuerzo de despliegue, configuración de redes, observabilidad y monitoreo para 1 administrador de sistemas y 4 programadores.
2. **Latencia de Comunicación Interna:** Tiempo de tránsito en la invocación de reglas de validación (red vs. memoria).
3. **Consistencia Transaccional:** Capacidad de ejecutar transacciones ACID locales sin requerir coordinación distribuida ni riesgo de estados inconsistentes.
4. **Mantenibilidad y Modificabilidad del Dominio:** Facilidad para cambiar reglas de negocio académicas sin provocar acoplamiento descontrolado (*spaghetti code*).

---

## 5. Matriz de Evaluación de Alternativas

| Criterio de Comparación | Alternativa 1: Microservicios | Alternativa 2: Monolito en Capas | Alternativa 3: Monolito Modular Hexagonal |
| :--- | :--- | :--- | :--- |
| **Carga Operativa (C10 / C08)** | **Muy Deficiente:** Requiere orquestadores (Kubernetes/Docker Swarm), mallas de servicio, observabilidad distribuida (tracing) y múltiples pipelines CI/CD. Inviable para 1 sysadmin. | **Excelente:** Un solo artefacto de despliegue en Linux. | **Excelente:** Un único artefacto principal + un proceso worker en Linux. Totalmente operable por el equipo actual. |
| **Latencia Interna (C02)** | **Deficiente:** Cada validación intermódulo agrega sobrecarga de red (serialización JSON/Protobuf, latencia TCP). | **Excelente:** Invocaciones en memoria local a velocidad de CPU (nanosegundos). | **Excelente:** Invocaciones en memoria entre módulos mediante interfaces públicas de dominio. Cero sobrecarga de red. |
| **Consistencia de Cupos (C04)** | **Deficiente:** Requiere transacciones distribuidas (Sagas o 2PC), introduciendo estados intermedios inconsistentes y ventanas de sobrecupo. | **Buena:** Transacciones ACID locales en PostgreSQL, pero propensa a bloqueos mutuos por falta de límites modulares. | **Excelente:** Transacciones ACID locales en el motor PostgreSQL, acotadas al módulo de inscripción sin dependencias de red. |
| **Mantenibilidad (DRV-10)** | **Buena:** Desacoplamiento físico de código. | **Muy Deficiente:** Alta tendencia a que los desarrolladores mezclen consultas y rompan reglas de negocio (*Big Ball of Mud*). | **Excelente:** Límites modulares claros forzados por arquitectura limpia. Dominio protegido de frameworks y persistencia. |

---

## 6. Decisión Seleccionada y Justificación Técnica

**Se decide adoptar la Alternativa 3: Monolito Modular con Arquitectura Limpia / Hexagonal.**

### Justificación:
1. **Evitar la trampa de la sobreingeniería:** En la industria y en entornos académicos existe la tendencia errónea de asumir que "más servicios equivalen a mejor arquitectura". Para la Universidad de la Salle, adoptar microservicios sería un error arquitectónico crítico: introduciría complejidad accidental masiva (gestión de fallas parciales, particionamiento de red, transacciones distribuidas) que desbordaría de inmediato la capacidad de 4 desarrolladores y 1 administrador (C10).
2. **Máxima eficiencia de hardware local (C08):** Al compilarse y ejecutarse en un único runtime en Linux, la PIA aprovecha al máximo la memoria RAM y los núcleos de CPU de los servidores disponibles, eliminando el consumo redundante de recursos que implican múltiples contenedores con runtimes duplicados.
3. **Consistencia atómica estricta (C04):** El control de cupos exige cero sobrecupos. Un monolito modular permite que el decremento del cupo y el registro de la inscripción ocurran dentro de una transacción local en PostgreSQL, eliminando la incertidumbre de sincronización entre servicios remotos.
4. **Independencia del dominio:** El uso de Arquitectura Hexagonal dentro de cada módulo garantiza que las reglas académicas (prerrequisitos, topes de créditos) sean clases de dominio puras, sin dependencias de librerías de terceros o del motor relacional, facilitando la cobertura de pruebas unitarias automatizadas por parte de los desarrolladores.

---

## 7. Consecuencias y Riesgos

### Consecuencias Positivas:
* **Simplicidad de despliegue:** Procedimiento de actualización directo mediante un servicio en Linux (`systemd` o contenedor Docker único), gestionable sin fricción por el administrador de infraestructura.
* **Depuración y trazabilidad simplificadas:** Los desarrolladores pueden reproducir errores y trazar el flujo completo de una inscripción en su entorno local con una única base de datos PostgreSQL, sin requerir emuladores de red compleja.
* **Cero costo de licenciamiento:** Cumple estrictamente con la restricción C08 al apoyarse 100% en software open source.

### Consecuencias Negativas y Desafíos:
* **Escalamiento horizontal acoplado:** Si el módulo de catálogo sufre una demanda extrema, se debe escalar la instancia completa del monolito (o desacoplar lecturas a nivel de réplica de base de datos) en lugar de escalar únicamente un microservicio aislado.
* **Disciplina rigurosa de desarrollo:** Existe el riesgo de que el equipo viole las fronteras entre módulos si no se aplican revisiones de código estrictas o analizadores estáticos de arquitectura (como ArchUnit o linters de módulos).

### Mitigación de Riesgos:
* Implementar reglas de empaquetado modular estrictas (visibilidad de paquetes restringida a interfaces públicas) y pruebas automatizadas de arquitectura en el pipeline de integración continua.
* Mitigar el escalamiento mediante el patrón CQRS ligero documentado en el diseño: descargar el 90% de las consultas de catálogo a la réplica de PostgreSQL y a la caché de Redis.

---

## 8. Condiciones para Revisar esta Decisión

Esta decisión arquitectónica deberá revisarse si y sólo si se presentan las siguientes circunstancias operativas:
1. El equipo de desarrollo institucional crece significativamente (más de 15 ingenieros) y se divide en equipos autónomos asignados por facultad o línea de negocio que requieran ciclos de despliegue totalmente independientes.
2. La universidad aprueba la migración integral hacia infraestructura en la nube pública y asigna presupuesto permanente para ingeniería DevOps y observabilidad distribuida.
