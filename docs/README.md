# Guía Técnica del Sistema de Liquidación Educativa – San Juan

> **Versión:** 1.0 — Octubre 2025
> **Autor:** Miguel
> **Revisado por:** ChatGPT (Asesor Senior – Arquitectura & Backend)

---

## 🧭 Índice

1. [Introducción](#introducción)
2. [Arquitectura General](#arquitectura-general)
3. [Diseño de Módulos](#diseño-de-módulos)

   * [Módulo 1: Legajos y Cargos (Core Data)](#módulo-1-legajos-y-cargos-core-data)
   * [Módulo 2: Parámetros Salariales (Configuration)](#módulo-2-parámetros-salariales-configuration)
   * [Módulo-3-liquidación-the-engine)](#módulo-3-liquidación-the-engine)
   * [Módulo 4: Integración y Reportes (Compliance & Output)](#módulo-4-integración-y-reportes-compliance--output)
   * [Módulo 5: Herramientas Auxiliares (Utils)](#módulo-5-herramientas-auxiliares-utils)
4. [Buenas Prácticas y Calidad de Código](#buenas-prácticas-y-calidad-de-código)
5. [Seguridad y Observabilidad](#seguridad-y-observabilidad)
6. [Testing y CI/CD](#testing-y-cicd)
7. [Trade-offs y Decisiones Clave](#trade-offs-y-decisiones-clave)
8. [Estructura de Carpetas](#estructura-de-carpetas)
9. [Preguntas Abiertas / Pendientes](#preguntas-abiertas--pendientes)

---

## 🧱 Introducción

Este documento define la **arquitectura técnica, diseño de módulos y estándares de desarrollo** para el proyecto **Sistema de Liquidación Educativa de San Juan**.

El objetivo es construir una **plataforma robusta, auditable y escalable** para la liquidación de sueldos docentes, con foco en:

* Cumplimiento normativo provincial y nacional (AFIP, LSD, Banco San Juan)
* Escalabilidad en cálculos masivos
* Auditoría total de resultados
* Mantenibilidad y extensibilidad a futuro

El diseño sigue los principios **SOLID**, el paradigma **DDD (Domain-Driven Design)** y las prácticas de ingeniería de software modernas.

---

## ⚙️ Arquitectura General

### Stack Tecnológico

| Componente            | Tecnología                        | Justificación                                                                              |
| --------------------- | --------------------------------- | ------------------------------------------------------------------------------------------ |
| **Backend**           | Node.js (LTS) + TypeScript        | Tipado fuerte y ecosistema maduro.                                                         |
| **Framework**         | NestJS                            | Arquitectura modular, inyección de dependencias, y alta testabilidad.                      |
| **Base de Datos**     | PostgreSQL                        | ACID, integridad referencial y soporte para JSONB.                                         |
| **ORM**               | Prisma                            | Tipado end-to-end y migraciones seguras.                                                   |
| **Colas/Jobs**        | BullMQ + Redis                    | Procesamiento asíncrono de liquidaciones masivas, generación de PDFs y archivos bancarios. |
| **Observabilidad**    | OpenTelemetry + Pino logs         | Métricas, trazas y auditoría de ejecución.                                                 |
| **Frontend**          | React + TypeScript                | Escalabilidad y rendimiento para el UI.                                                    |
| **Testing**           | Jest (unitario), Playwright (E2E) | Cobertura total con entorno controlado.                                                    |
| **Contenedorización** | Docker                            | Entornos consistentes Dev/Staging/Prod.                                                    |

### Patrón Arquitectónico

**Monolito Modular** (MVP), con posibilidad de migrar a microservicios por dominio:

* `core/` (agentes, designaciones, servicios previos)
* `engine/` (cálculo, conceptos, fórmulas)
* `output/` (reportes, AFIP, banco, PDFs)

---

## 🧩 Diseño de Módulos

### Módulo 1: Legajos y Cargos (Core Data)

**Objetivo:** Fuente única de verdad (SSOT) para agentes, designaciones y servicios.

#### Decisiones Clave

* Cada `Designacion` posee un **ID inmutable** con `vigenteDesde/vigenteHasta`.
* Se incluye un `HistorialMovimientoDesignacion` (motivo, fecha, tipo de cambio) para trazabilidad completa.
* `AntiguedadService` calcula dinámicamente los años efectivos a la fecha de liquidación.

#### Modelo de Datos (Prisma simplificado)

```prisma
model Agente {
  id           String   @id @default(uuid())
  cuit         String   @unique
  dni          String   @unique
  apellido     String
  nombre       String
  designaciones Designacion[]
  serviciosPrevios ServicioPrevio[]
  movimientos   HistorialMovimientoDesignacion[]
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}
```

---

### Módulo 2: Parámetros Salariales (Configuration)

**Objetivo:** Desacoplar la lógica de cálculo de los valores monetarios.

#### Estructura

* `ParametroBase`: versión por `fecha_vigencia` con bundle atómico (índice docente, valor punto, topes, etc.)
* `Concepto`: definición de cada ítem (código interno, AFIP, tipo, fórmula, prioridad).
* `ReglaDeCalculo`: puntero a función implementada en el Engine.
* Validación de archivos de configuración con **Zod** o **JSON Schema**.

#### Mejora

* Permitir parámetros **transitorios** (bonos únicos o excepciones) con `vigenteHasta` puntual.

---

### Módulo 3: Liquidación (The Engine)

**Objetivo:** Lógica de negocio pura, determinista y testeable.

#### Flujo General

1. Cargar agente + designaciones + novedades a fecha.
2. Obtener parámetros vigentes.
3. Calcular base (puntos × índice).
4. Ejecutar conceptos mediante funciones puras (`rules/`).
5. Consolidar resultados y registrar auditoría.

#### Interfaz Base

```ts
export interface LiquidacionResult {
  agenteId: string;
  periodo: string;
  items: ItemCalculado[];
  totales: { remunerativo: Decimal; noRemunerativo: Decimal; descuentos: Decimal; neto: Decimal; };
  trazabilidad: { parametrosVigentesId: string; hashInput: string; };
}
```

#### Mejores Prácticas Incorporadas

* **Rollback transaccional** por agente.
* **Idempotencia** (no duplicar liquidaciones).
* **Memoización** interna en cálculos repetitivos.
* **Cache** de antigüedad y parámetros vigentes.
* **Rastreo** con hash único y trazas OpenTelemetry.

---

### Módulo 4: Integración y Reportes (Compliance & Output)

**Objetivo:** Cumplimiento legal y salida de datos.

* `LsdGeneratorService`: genera archivos AFIP (TXT/JSON) validados con JSON Schema.
* `ArchivoBancarioService`: drivers por banco (interfaz `BankAdapter`).
* `RecibosService`: genera PDF firmados digitalmente (Puppeteer + hash SHA-256 + firma X.509).
* **Pre-flight validator:** chequea totales y formatos antes de emitir archivos.

---

### Módulo 5: Herramientas Auxiliares (Utils)

**Objetivo:** Apoyo funcional sin impacto en la corrida mensual.

| Herramienta                         | Descripción                                | Implementación                         |
| ----------------------------------- | ------------------------------------------ | -------------------------------------- |
| **Simulador de Cargos**             | Calcula diferencias entre cargos o radios. | Usa el Engine en modo simulación.      |
| **Reconocimiento de Servicios**     | CRUD de servicios docentes previos.        | Alimenta al `AntiguedadService`.       |
| **Comparador por Radio/Antigüedad** | Muestra impacto directo de variaciones.    | UI con export CSV.                     |
| **Validador Normativo**             | Detecta inconsistencias administrativas.   | Reglas automáticas previas a liquidar. |

---

## 🧩 Buenas Prácticas y Calidad de Código

* **SOLID:** Cada servicio con una sola responsabilidad.
* **Extensibilidad:** Nuevos conceptos agregables sin tocar el core (`Open/Closed`).
* **Inyección de Dependencias:** Usar interfaces abstractas para persistencia y servicios.
* **Manejo de Errores:** Clases específicas + middleware global.
* **Documentación:** Uso de JSDoc y `Typedoc` o `Compodoc`.
* **Feature Flags:** activar nuevos conceptos sin redeploy.

---

## 🔒 Seguridad y Observabilidad

| Área                           | Estrategia                                                            |
| ------------------------------ | --------------------------------------------------------------------- |
| **Autenticación/Autorización** | RBAC con guardias NestJS + scopes por institución (multi-tenant).     |
| **Validación de Input**        | DTOs con `class-validator` y sanitización.                            |
| **Secretos**                   | Variables de entorno gestionadas por `@nestjs/config`.                |
| **Auditoría**                  | Tabla `AuditLog` con timestamp, usuario, acción y entidad modificada. |
| **Cifrado de Datos**           | CUIT/DNI cifrados con AES-GCM.                                        |
| **Observabilidad**             | OpenTelemetry + Pino logs estructurados.                              |
| **Rate Limiting**              | Control en endpoints de descargas (PDF/TXT).                          |

---

## 🧪 Testing y CI/CD

### Estrategia de Testing

* **Unitario (Jest):** 100% cobertura en `engine/rules`.
* **Property-based Testing:** con `fast-check` para redondeos y prorrateos.
* **Integración:** Motor + DB (Prisma test env).
* **E2E (Playwright):** flujos críticos (crear agente → correr liquidación → generar LSD/PDF).
* **Fixtures:** escenarios reales anonimizados.

### CI/CD Pipeline

1. Lint (ESLint) + formato (Prettier).
2. Type-check.
3. Unit + Integration Tests.
4. Build Docker multi-stage.
5. Deploy a **staging** con seeders automáticos.
6. E2E contra staging.
7. Deploy a **prod** con migraciones transaccionales.

### Seeds y Versionado

```bash
npm run seed:test
npm run seed:staging
```

Versiones etiquetadas semánticamente (`v1.0.0-liquidacion`, `v1.1.0-LSD`).

---

## ⚖️ Trade-offs y Decisiones Clave

| Tema                | Decisión                                             | Justificación                                        |
| ------------------- | ---------------------------------------------------- | ---------------------------------------------------- |
| **Persistencia**    | Opción B: Detalle por ítem (`ItemLiquidacion`).      | Auditoría total y generación directa de recibos/LSD. |
| **Fórmulas**        | Opción A: Implementadas en código TypeScript.        | Seguridad y testabilidad.                            |
| **Ganancias**       | Solo retención automática (sin F.572).               | Simplifica MVP.                                      |
| **Infraestructura** | Monolito modular con posibilidad de escisión futura. | Equilibrio entre complejidad y mantenibilidad.       |

---

## 📁 Estructura de Carpetas

```
/apps/api
  /src
    /modules
      agentes/
      designaciones/
      parametros/
      conceptos/
      liquidacion/
        engine/
          rules/
          services/
          dto/
        repositories/
      reportes/
      integraciones/
        afip/
        bancos/
      recibos/
    /shared
      /domain
      /infra
      /utils
  /test
    /unit
    /integration
    /e2e
/prisma
  schema.prisma
```

---

## ❓ Preguntas Abiertas / Pendientes

1. ¿El sistema debe soportar **multi-institución** en un mismo período con parámetros diferentes?
2. ¿Habrá **estados de liquidación** (BORRADOR → VALIDADA → FINAL)?
3. ¿Confirmar uso de **certificado X.509** para firma digital?
4. ¿Layouts adicionales además de Banco San Juan?
5. ¿Manejo de **retroactivos** paritarios? (Re-liquidaciones automáticas).

---

> *Esta guía sirve como base técnica oficial para el desarrollo del MVP y las futuras evoluciones del Sistema de Liquidación Educativa – San Juan.*
