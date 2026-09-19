# PRD – BecasDSS: Sistema de Gestión y Análisis de Becas Estudiantiles

| Campo | Valor |
|---|---|
| **Squad** |  Becasdss  |
| **Repositorio** |  https://github.com/alexisaguirreflores20-blip/BecasDSS  |
| **Versión** | 1.1 |
| **Fecha** | 18/09/2026 |
| **Materia** | Ingeniería en Sistemas – UPDS Tarija |

---

# 1. Objetivo y Fronteras (Boundary Rules)

## 1.1 Propósito

**BecasDSS** es un sistema de dos capas para una institución educativa que otorga becas estudiantiles. La primera capa (**OLTP/CRUD**) registra y controla todo el ciclo de una beca: convocatorias, postulaciones, evaluaciones, asignaciones y desembolsos. La segunda capa (**OLAP/DSS**) consolida esa información en un Data Warehouse y ofrece KPIs y dashboards para que las autoridades decidan con datos: cuántas becas asignar, a qué carreras, con qué presupuesto y con qué resultados.

## 1.2 Lo que DEBE hacer

- **Gestionar el ciclo completo de una beca** (convocatoria → postulación → evaluación → asignación → desembolso) con trazabilidad de cada cambio.
- **Calcular y aplicar de forma determinista la asignación de becas** según puntaje y cupos, sin intervención manual sobre el orden de prelación.
- **Alimentar un Data Warehouse mediante ETL diario** y exponer KPIs de aprobación, ejecución presupuestaria y tiempos de resolución.

## 1.3 Lo que NO debe hacer / tocar (Boundary Rules)

> Estas reglas aplican por igual a programadores humanos y agentes de IA. Ante cualquier ambigüedad, **el agente debe detenerse y preguntar**; nunca inferir.

| ID | Regla |
|---|---|
| **BR-01** | **NO** borrar físicamente registros de postulaciones, evaluaciones, asignaciones ni desembolsos. Solo baja lógica (`activo = 0`) con motivo obligatorio. |
| **BR-02** | **NO** modificar, truncar ni eliminar las tablas de seguridad y auditoría (`usuario`, `rol`, `permiso`, `auditoria`). |
| **BR-03** | **NO** escribir en el OLTP desde el módulo DSS. El DSS es de **solo lectura** sobre el Data Warehouse; jamás consulta el OLTP directamente. |
| **BR-04** | **NO** exponer datos personales (CI, teléfono, correo, dirección) en dashboards ni exportaciones. El DW almacena el CI únicamente como hash SHA-256. |
| **BR-05** | **NO** alterar el esquema de la base de datos (DDL) ni scripts de migración ya aplicados. Todo cambio de esquema se hace con un **nuevo** script versionado y revisado. |
| **BR-06** | **NO** incluir credenciales, cadenas de conexión ni claves en el repositorio. Solo variables de entorno; `.env` está en `.gitignore`. |
| **BR-07** | **NO** hacer commit directo a `main`. Todo cambio entra por Pull Request con al menos 1 aprobación. |
| **BR-08** | **NO** cambiar reglas de negocio (puntajes, umbrales, cupos) sin una historia de usuario aprobada que lo indique. |
| **BR-09** | **NO** integrar pasarelas de pago, bancos ni sistemas externos. Los desembolsos son **registros contables**, no transferencias reales. |
| **BR-10** | Los agentes de IA solo pueden modificar archivos dentro de `/src`, `/tests` y `/docs`. Cualquier otra ruta requiere autorización humana explícita. |
| **BR-11** | **Privacidad y retención:** los datos personales (nombre, CI, teléfono, correo, dirección) de postulaciones `Rechazada` o `Anulada` se **anonimizan** a los **24 meses** del cierre de la convocatoria (se reemplazan por valores nulos; es una actualización, no un borrado). Desembolsos y auditoría se conservan sin datos personales. Ningún agente ni programador puede alterar este plazo sin una historia aprobada. |
| **BR-12** | **Entorno:** los agentes de IA y los scripts automáticos solo pueden ejecutar operaciones contra bases de datos de **desarrollo o pruebas**. Está prohibido conectarse a una base de datos de producción. |

---

# 2. Perfiles de Usuario

| Perfil | Módulo | Descripción |
|---|---|---|
| **Administrador de Bienestar Estudiantil** | CRUD | Crea convocatorias, gestiona postulaciones, asigna becas y registra desembolsos. |
| **Estudiante Postulante** | CRUD | Registra su postulación, adjunta documentos y consulta su estado. |
| **Miembro del Comité Evaluador** | CRUD | Registra evaluaciones y puntajes de las postulaciones asignadas. |
| **Director / Autoridad Académica** | DSS | Consulta KPIs y dashboards para decidir cupos y presupuesto. |
| **Analista de Datos** | DSS | Filtra, cruza y exporta información agregada; monitorea el ETL. |

## Matriz de permisos (CRUD)

| Operación | Administrador | Estudiante | Comité |
|---|:---:|:---:|:---:|
| Crear/editar convocatoria | ✅ | ❌ | ❌ |
| Registrar postulación | ❌ | ✅ (solo la propia) | ❌ |
| Consultar postulación | ✅ (todas) | ✅ (solo la propia) | ✅ (asignadas) |
| Registrar evaluación | ❌ | ❌ | ✅ |
| Asignar beca | ✅ | ❌ | ❌ |
| Registrar desembolso | ✅ | ❌ | ❌ |
| Anular (baja lógica) | ✅ | ❌ | ❌ |

---

# 3. Módulo 1: Requerimientos Transaccionales (CRUD / OLTP)

## 3.1 Épicas

- **E1 – Gestión de Convocatorias:** definir períodos, cupos y presupuesto.
- **E2 – Postulaciones:** registro, documentos y seguimiento por el estudiante.
- **E3 – Evaluación y Asignación:** puntuar y adjudicar becas de forma objetiva.
- **E4 – Desembolsos:** control del dinero entregado a cada becado.

## 3.2 Historias de Usuario

### HU-01 (E1) – Crear convocatoria
**Como** Administrador de Bienestar **quiero** crear una convocatoria con nombre, gestión académica, tipo de beca, cupos, presupuesto, fecha de inicio y fecha de cierre **para** que los estudiantes puedan postular solo dentro del período habilitado.

**Criterios de aceptación**
- Dado que la fecha de cierre es anterior o igual a la de inicio, cuando guardo, entonces el sistema rechaza con el mensaje "La fecha de cierre debe ser posterior a la de inicio".
- Cupos es un entero ≥ 1 y presupuesto es un decimal > 0; de lo contrario se rechaza el registro.
- Al guardar exitosamente, la convocatoria queda en estado `Abierta` si la fecha actual está dentro del rango, o `Programada` si es futura.

### HU-02 (E2) – Registrar postulación
**Como** Estudiante Postulante **quiero** completar el formulario de postulación y adjuntar mis documentos **para** participar en una convocatoria abierta.

**Criterios de aceptación**
- Solo se permite **una** postulación por estudiante por convocatoria (validación en base de datos con restricción `UNIQUE`).
- Cada documento debe ser PDF y pesar ≤ 5 MB; en caso contrario se rechaza indicando el archivo.
- Fuera del plazo de la convocatoria el sistema bloquea el envío.
- Al registrarse, la postulación queda en estado `Registrada` y el estudiante recibe un código de seguimiento.

### HU-03 (E2) – Consultar estado de postulación
**Como** Estudiante Postulante **quiero** ver el estado actual de mi postulación (`Registrada`, `En evaluación`, `Aprobada`, `Rechazada`, `Anulada`) **para** saber en qué etapa está sin acudir a la oficina.

**Criterios de aceptación**
- El estudiante solo ve sus propias postulaciones.
- Si está `Rechazada`, se muestra el motivo: "Puntaje inferior a 70" o "Cupos agotados".

### HU-04 (E3) – Registrar evaluación
**Como** Miembro del Comité Evaluador **quiero** registrar el puntaje de una postulación por criterio (rendimiento académico 0–40, situación socioeconómica 0–40, entrevista 0–20) **para** obtener un puntaje total objetivo de 0 a 100.

**Criterios de aceptación**
- El sistema rechaza puntajes fuera del rango de cada criterio.
- El puntaje total se calcula automáticamente como la suma de los tres criterios; no es editable manualmente.
- Cada postulación es evaluada por **un solo** miembro del Comité (el primero en registrarla); el sistema bloquea una segunda evaluación.
- Una corrección solo la autoriza el Administrador anulando la evaluación con motivo (baja lógica); luego se registra de nuevo.
- Registrar la evaluación **no** cambia el estado: la postulación sigue `En evaluación` hasta la asignación (HU-05).

### HU-05 (E3) – Asignar becas
**Como** Administrador de Bienestar **quiero** ejecutar la asignación de una convocatoria cerrada **para** adjudicar becas de forma automática y sin sesgo.

**Criterios de aceptación**
- Solo se ejecuta si la convocatoria ya cerró y **todas** las postulaciones `En evaluación` tienen evaluación; si falta alguna, se bloquea y se lista cuáles.
- Solo son elegibles las postulaciones con puntaje total ≥ 70.
- Se adjudica en orden descendente de puntaje total hasta agotar los cupos.
- **Desempate:** primero el mayor puntaje del criterio socioeconómico; si persiste, el mayor puntaje de rendimiento académico; si persiste, la postulación registrada primero (fecha y hora).
- Las postulaciones adjudicadas pasan a `Aprobada`; las restantes pasan a `Rechazada` con motivo "Puntaje inferior a 70" o "Cupos agotados".
- La asignación se ejecuta una sola vez por convocatoria y queda registrada en auditoría.

### HU-06 (E4) – Registrar desembolso
**Como** Administrador de Bienestar **quiero** registrar los desembolsos mensuales de un becado **para** controlar cuánto se ha entregado frente al monto aprobado.

**Criterios de aceptación**
- La suma de desembolsos de un becado no puede superar el monto total aprobado.
- No se permite registrar dos desembolsos del mismo mes para el mismo becado.
- Un desembolso solo puede registrarse para becas en estado `Aprobada`.

### HU-07 (E2/E3) – Anular postulación
**Como** Administrador de Bienestar **quiero** anular una postulación indicando el motivo **para** corregir errores o fraudes sin perder el historial.

**Criterios de aceptación**
- El motivo es obligatorio (mínimo 10 caracteres).
- La anulación es una baja lógica (BR-01) y queda en auditoría con usuario, fecha y hora.
- No se puede anular una postulación con desembolsos registrados.

## 3.3 Máquina de estados de la postulación

| Desde | Hacia | Lo ejecuta | Condición |
|---|---|---|---|
| *(nueva)* | `Registrada` | Estudiante | HU-02 |
| `Registrada` | `En evaluación` | Sistema | Automático cuando la convocatoria alcanza su fecha de cierre |
| `Registrada` | `Anulada` | Administrador | Motivo obligatorio (HU-07) |
| `En evaluación` | `Aprobada` | Sistema | Asignación (HU-05) |
| `En evaluación` | `Rechazada` | Sistema | Asignación (HU-05) |
| `En evaluación` | `Anulada` | Administrador | Motivo obligatorio (HU-07) |
| `Aprobada` | `Anulada` | Administrador | Solo si no tiene desembolsos (HU-07) |

**Transiciones prohibidas:** cualquier salida desde `Rechazada` o `Anulada` (son estados finales); `Aprobada → Rechazada`; `Registrada → Aprobada`; `Registrada → Rechazada`. El agente debe bloquear y reportar cualquier intento.

---

# 4. Módulo 2: Requerimientos Analíticos (DSS / OLAP)

## 4.1 Integración (ETL)

- **Extracción:** cada día a las **02:00** se extraen del OLTP solo los registros nuevos o modificados desde la última ejecución (carga incremental por columna `fecha_modificacion`).
- **Transformación:** limpieza de nulos, estandarización de nombres de carrera, cálculo de campos derivados (días de resolución, monto ejecutado) y **anonimización del CI mediante hash SHA-256**.
- **Carga:** inserción en un **esquema estrella** del Data Warehouse.
  - **Hechos:** `fact_postulacion`, `fact_desembolso`.
  - **Dimensiones:** `dim_tiempo`, `dim_carrera`, `dim_convocatoria`, `dim_tipo_beca`, `dim_estudiante` (anonimizada).
- **Control:** cada ejecución registra en `etl_log` los conteos de origen y destino, la duración y el estado. Si los conteos no coinciden, la carga se marca como fallida y no se publica.

## 4.2 Historias de Usuario DSS

### HU-DSS-01 – Tasa de aprobación
**Como** Director **quiero** ver la tasa de aprobación (`aprobadas / postulaciones evaluadas × 100`) por convocatoria y por carrera **para** identificar carreras con baja adjudicación y ajustar cupos.

**Criterios de aceptación:** se muestra en gráfico de barras con valores en porcentaje con 1 decimal; permite comparar hasta 4 gestiones académicas.

### HU-DSS-02 – Ejecución presupuestaria
**Como** Director **quiero** ver el porcentaje de presupuesto ejecutado (`desembolsado / presupuesto aprobado × 100`) por convocatoria y por mes **para** detectar sobrantes o riesgo de déficit.

**Criterios de aceptación:** el indicador se pinta en rojo si la ejecución supera el 100% o es menor al 60% al cierre de la convocatoria.

### HU-DSS-03 – Tiempo de resolución
**Como** Analista de Datos **quiero** ver el promedio de días entre el registro de la postulación y la fecha de asignación de su convocatoria (HU-05, que es el dictamen final) **para** detectar cuellos de botella en el proceso.

**Criterios de aceptación:** se marca una alerta cuando el promedio supera **15 días calendario**; se puede desglosar por convocatoria.

### HU-DSS-04 – Filtros y exportación
**Como** Analista de Datos **quiero** filtrar por gestión, carrera, tipo de beca y estado, y exportar el resultado a CSV **para** elaborar informes institucionales.

**Criterios de aceptación:** la exportación no incluye datos personales (BR-04) y se limita a 50.000 filas por archivo.

---

# 5. Requerimientos No Funcionales

| ID | Categoría | Requisito medible |
|---|---|---|
| **RNF-01** | Rendimiento (CRUD) | El percentil 95 del tiempo de respuesta de operaciones CRUD individuales debe ser **≤ 400 ms** con 20.000 postulaciones almacenadas y 50 usuarios concurrentes. |
| **RNF-02** | Rendimiento (DSS) | Cada dashboard debe cargar en **≤ 3 s (percentil 95)** consultando hasta 200.000 filas en tablas de hechos. |
| **RNF-03** | ETL | La carga incremental diaria de hasta 5.000 registros debe completarse en **≤ 10 min**, con **100 %** de coincidencia entre conteos de origen y destino. |
| **RNF-04** | Disponibilidad | **≥ 99,5 %** mensual entre las 06:00 y las 22:00 (tolerancia máxima ≈ 2,4 h de caída al mes). |
| **RNF-05** | Seguridad | Contraseñas almacenadas con **bcrypt (cost ≥ 12)**; sesión expira a los **30 min** de inactividad; **100 %** de operaciones de escritura quedan en auditoría (usuario, fecha/hora UTC, valor anterior y nuevo). |
| **RNF-06** | Calidad de código | Cobertura de pruebas unitarias **≥ 80 %** en la lógica de negocio (puntaje, asignación, desembolsos). |
| **RNF-07** | Tasa de errores | Menos de **1 %** de respuestas con error 5xx durante una prueba de carga de 10 min con 50 usuarios concurrentes. |
| **RNF-08** | Recuperación | Respaldo completo diario a las **01:00**; **RPO ≤ 24 h** y **RTO ≤ 4 h**, verificados con **1** restauración de prueba exitosa al mes. |

**Método de medición:** las pruebas de carga se ejecutan con **k6** (o JMeter) contra el entorno de pruebas del squad, con el mismo motor y esquema que producción y datos sintéticos con el volumen indicado en cada RNF. Los umbrales se recalibran después de la primera medición del prototipo y el resultado se registra en `/docs`.

---

# 6. Puertas de Calidad (Quality Gates)

## 6.1 DoR – Definition of Ready (para empezar a programar)

- [ ] La historia sigue el formato **Como [rol] quiero… para…** y cumple **INVEST**.
- [ ] Tiene **criterios de aceptación** verificables (Dado / Cuando / Entonces).
- [ ] Las reglas de negocio involucradas están escritas (rangos, umbrales, desempates).
- [ ] Se identificaron las tablas OLTP o del DW afectadas y no se viola ninguna Boundary Rule (BR-01 a BR-12).
- [ ] Está estimada (puntos de historia) y cabe en un sprint.
- [ ] Tiene dependencias resueltas y un responsable asignado en GitHub Projects.

## 6.2 DoD – Definition of Done (para aceptar la funcionalidad)

- [ ] Cumple **todos** los criterios de aceptación de la historia.
- [ ] Pruebas unitarias aprobadas y cobertura **≥ 80 %** en la lógica nueva.
- [ ] Ninguna Boundary Rule fue violada (revisado en el PR).
- [ ] Los RNF aplicables fueron medidos y cumplen los umbrales de la sección 5.
- [ ] Pull Request revisado y aprobado por al menos 1 integrante distinto al autor.
- [ ] Sin credenciales ni datos personales en el código o en los logs.
- [ ] Documentación actualizada en `/docs` y tarjeta movida a *Done* en GitHub Projects.

---

# 7. Mapeo de Arquitectura

```text
+------------------+      +--------------------+      +----------------------+
|   [OLTP / CRUD]  |      |        [ETL]       |      |    [OLAP / DSS]      |
|                  |      |                    |      |                      |
| Convocatorias    |      | Extract  (02:00,   |      | Data Warehouse       |
| Postulaciones    | ---> |  incremental)      | ---> |  (esquema estrella)  |
| Evaluaciones     |      | Transform (limpieza|      |                      |
| Asignaciones     |      |  + hash CI)        |      | KPIs y Dashboards    |
| Desembolsos      |      | Load  (+ etl_log)  |      | (solo lectura)       |
+------------------+      +--------------------+      +----------------------+
   ^ Administrador,                                       ^ Director,
   Estudiante, Comité                                       Analista de Datos

Regla de oro: el flujo es unidireccional  OLTP --> ETL --> OLAP
El DSS nunca escribe ni consulta el OLTP (BR-03).
```

---

# 8. Supuestos y Decisiones Pendientes

- Los umbrales de la sección 5 y los volúmenes (20.000 postulaciones, 200.000 filas de hechos) son **estimaciones de diseño**; deben ajustarse tras medir el prototipo.
- Se asume un motor relacional (p. ej. SQL Server 2022) tanto para OLTP como para el Data Warehouse.
- Los pesos de puntaje (40/40/20) y el umbral mínimo de 70 puntos son reglas propuestas, sujetas a la normativa de becas de la institución.
- El plazo de retención de 24 meses (BR-11) y los valores de RPO/RTO (RNF-08) son propuestas del squad, sujetas a validación institucional.

---

# 9. Historial de Cambios

| Versión | Fecha | Cambios |
|---|---|---|
| 1.0 | 18/09/2026 | Borrador inicial del PRD. |
| 1.1 | 18/09/2026 | Correcciones del Dictamen de Verificación: (1) máquina de estados de la postulación, evaluador único y definición de "dictamen final"; (2) reglas BR-11 (privacidad y retención) y BR-12 (solo entornos de desarrollo/pruebas); (3) método de medición de RNF y nuevo RNF-08 de recuperación. |
