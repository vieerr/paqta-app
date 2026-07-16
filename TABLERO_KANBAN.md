# Tablero Kanban — Proyecto de Microservicios

## Cómo crearlo en GitHub Projects (5 min)
1. Repo → pestaña **Projects** → **New project** → plantilla **Board**.
2. Columnas: **Backlog · Por hacer · En progreso · En revisión · Hecho**.
3. Creen las tarjetas de abajo. A cada una: **responsable** + etiqueta de avance (`avance-1/2/3`).
4. Muevan las tarjetas conforme avanzan. Al cerrar cada avance suban **una captura** a `/docs` y enlácenla en el README.

> Si no usan GitHub Projects, usen la tabla Markdown del final como tablero dentro del repo.

---

## Tarjetas iniciales

### 🟢 Avance 1 — `avance-1` (documentación de paqta real, sin infra nueva)
- [x] Confirmar dominio: 6 microservicios reales (auth, agency, billing, files, proposals, magic-links)
- [x] Crear repo hub `paqta-app` (solo docs), rama `main`
- [x] Auditar los 6 repos: transporte, persistencia, patrones (sin cambios de código)
- [x] Confirmar ausencia de TCP/Redis/SQS/SNS/EventBridge en el sistema real (búsqueda exhaustiva)
- [x] Identificar el único camino síncrono multi-salto real (`delete-account.ts` → 4 servicios)
- [x] Identificar el acoplamiento real no mitigado (Lambda Authorizer compartido vía SSM)
- [ ] ~~Camino asíncrono con Redis~~ — no existe en paqta real, documentado como brecha vs. la guía
- [x] Manejo de excepciones — evidenciado con citas (`delete-account.ts`, `dodo-client.ts`)
- [x] Benchmark de latencia real contra endpoints en vivo (`magic-links-api-dev`, `auth-api-dev`)
- [x] Análisis de acoplamiento temporal basado en código + comportamiento observado (sin apagar prod)
- [x] Diagrama de arquitectura real (Mermaid) + README Avance 1
- [x] Tag `v1-avance1`

### 🟡 Avance 2 — `avance-2`
- [ ] Definir contrato `.proto` (gRPC) entre dos microservicios
- [ ] Implementar comunicación gRPC en el monorepo
- [ ] try/catch para controlar errores en gRPC
- [ ] Agregar segundo transporte (RabbitMQ/MQTT/NATS) con PUB/SUB o queue
- [ ] Demostrar error controlado sin caída del servicio
- [ ] Tabla comparativa de transportes
- [ ] Diagrama actualizado + README Avance 2
- [ ] Tag `v2-avance2`

### 🔵 Avance 3 — `avance-3`
- [ ] Login que emite token JWT
- [ ] Validación del JWT en las rutas
- [ ] Guard que protege rutas (401 sin token; 403 por rol si aplica)
- [ ] Integrar logs con Sentry (capturar errores)
- [ ] Integrar todos los microservicios/transportes en una operación
- [ ] Diagrama final + README Avance 3 + sección Defensa
- [ ] Preparar diapositivas y ensayar demo
- [ ] Tag `v3-final`

---

## Tablero Markdown (alternativa dentro del repo)
| Backlog | Por hacer | En progreso | En revisión | Hecho |
|---|---|---|---|---|
| Integrar Sentry (T3) | Contrato gRPC (T2) | — | — | Auditoría de los 6 repos reales |
| JWT + Guard (T3) | Segundo transporte (T2) | — | — | README Avance 1 + diagrama Mermaid |
| | | | | Benchmark real (magic-links-api-dev, auth-api-dev) |
| | | | | Análisis de acoplamiento (Lambda Authorizer compartido) |
| | | | | Tag `v1-avance1` |
