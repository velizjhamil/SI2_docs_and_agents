# SI2 — Plataforma Transaccional de Gestión Integral para Cooperativas

## Descripción
Plataforma web y móvil que digitaliza la afiliación de socios y solicitud de créditos en cooperativas.
Incluye onboarding digital con visión artificial (OCR de carnet/boleta) y un asistente de precalificación
crediticia mediante lenguaje natural.

## Repositorios

| Repo | Descripción | Tecnología |
|---|---|---|
| `SI2_docs_and_agents` | Specs, documentación y configuración de agentes | OpenSpec, Engram |
| `SI2_backend_api` | API REST principal | Python, FastAPI, PostgreSQL |
| `SI2_frontend_web` | Aplicación web | Angular (TypeScript) |
| `SI2_mobile_app` | Aplicación móvil | Flutter (Dart) |

## Stack Técnico

- **Backend:** FastAPI (Python), PostgreSQL
- **Frontend Web:** Angular
- **Mobile:** Flutter + Dart
- **IA / OCR:** Visión artificial para extracción de datos desde imágenes de documentos
- **IA / Chat:** LLM para asistente conversacional de precalificación
- **Memoria de agentes:** Engram (SQLite + MCP)
- **Specs versionadas:** OpenSpec

## Módulos Principales

### 1. Onboarding Digital
- Registro de nuevos socios
- Captura y OCR de carnet de identidad y boleta de pago
- Extracción automática: nombre, CI, ingresos
- Validación y almacenamiento estructurado

### 2. Motor de Riesgo Crediticio
- Cálculo matemático del límite de crédito por socio
- Lógica basada en reglas e indicadores financieros
- Sin dependencia de IA para el cálculo central

### 3. Asistente de Precalificación (Chat IA)
- Recibe el resultado numérico del motor de riesgo
- Comunica la precalificación al socio de forma conversacional
- Explica condiciones, pasos y alternativas

### 4. Gestión de Créditos
- Solicitud, seguimiento y aprobación de créditos
- Panel administrativo para gestores de la cooperativa

### 5. Autenticación y Roles
- Roles: socio, asesor, administrador
- JWT + refresh tokens

## Convenciones

- Las specs se escriben en `openspec/changes/` de este repo
- Cada feature cross-repo documenta qué archivos toca en cada repositorio
- Los cambios de base de datos siempre incluyen migración en la spec
- El agente (OpenCode) debe guardar en Engram toda decisión de arquitectura relevante

## Reglas para el Agente

- Antes de implementar cualquier feature, revisar si existe una spec en `openspec/changes/`
- No modificar el motor de riesgo sin spec aprobada explícitamente
- Las rutas de la API siguen el patrón `/api/v1/{módulo}/{recurso}`
- Los modelos de base de datos van en `SI2_backend_api/app/models/`
- Los servicios de IA son externos al motor de riesgo y nunca alteran su lógica