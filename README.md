# paqta — Proyecto Integrador de Microservicios (hub de documentación)

> MVP de arquitectura de microservicios · Arquitectura de Software / Microservicios · 7.° semestre · Entrega por avances.
>
> Este repo **no contiene código propio**: es el hub de documentación del proyecto real **paqta**
> (SaaS de gestión de agencias y propuestas). Todo lo descrito aquí se verificó leyendo el código
> fuente real y, cuando fue posible sin acciones destructivas, midiendo los endpoints reales
> desplegados en AWS (stage `dev`).

## 👥 Equipo
| Integrante | Rol | GitHub |
|---|---|---|
| Olivier Paspuel | Backend / Arquitectura | [@vieerr](https://github.com/vieerr) |
| <<Nombre 2>> | Transportes / gRPC | @usuario |
| <<Nombre 3>> | Seguridad / Observabilidad | @usuario |
| <<Nombre 4 (opcional)>> | Documentación / QA | @usuario |

## 🔗 Repositorios reales
| Repo | Responsabilidad |
|---|---|
| [paqta](https://github.com/vieerr/paqta) | Frontend Next.js — orquesta las llamadas HTTP a cada microservicio (topología en estrella) |
| [pq-auth-ms](https://github.com/vieerr/pq-auth-ms) | Identidad (Clerk), autorizador Lambda compartido, borrado de cuenta |
| [pq-agency-ms](https://github.com/vieerr/pq-agency-ms) | Perfil/onboarding de agencias |
| [pq-billing-ms](https://github.com/vieerr/pq-billing-ms) | Suscripciones/entitlements (Dodo Payments) |
| [pq-files-ms](https://github.com/vieerr/pq-files-ms) | Subida/descarga de archivos (S3) + extracción de estilo por IA |
| [pq-proposals-ms](https://github.com/vieerr/pq-proposals-ms) | Generación de propuestas (Gemini AI), export PDF |
| [pq-magic-link-ms](https://github.com/vieerr/pq-magic-link-ms) | Enlaces públicos de propuestas + tracking de apertura (Resend) |

## 🧱 Stack real
- **Framework:** AWS SST (Serverless Stack) — cada microservicio es un conjunto de funciones **AWS Lambda**
  detrás de **API Gateway HTTP API**, no procesos NestJS/Express de larga duración.
- **Persistencia:** 100% **DynamoDB** (`sst.aws.Dynamo` en cada `sst.config.ts`). No hay TypeORM ni Postgres
  en el sistema real.
- **Auth:** Clerk + un **Lambda Authorizer compartido**, publicado por `pq-auth-ms` vía SSM Parameter Store
  y consumido por los demás servicios (ver §Acoplamiento).
- **Transporte entre servicios:** HTTPS (`fetch`) directo a las URLs de API Gateway de cada servicio — **no
  hay TCP crudo, ni gRPC, ni cola/broker de eventos (SQS/SNS/EventBridge/Redis) en ningún repo real.**

> ⚠️ **Esto difiere del enunciado de la Tarea 1**, que pide un camino síncrono por **TCP** y uno asíncrono
> por **eventos (Redis)**. Se documenta abajo, con evidencia de código, **exactamente lo que existe** en
> paqta real — sin simular una infraestructura que no está desplegada. La sección "🧠 Análisis" explica
> por qué y qué implica para la rúbrica.

## ▶️ Cómo verificar (endpoints reales, stage `dev`)
```bash
# Endpoint público real (sin auth) — usado para medir latencia
curl https://magic-links-api-dev.paqta.app/api/v1/magic-links/<token>/public

# Endpoint protegido real (pasa por el Lambda Authorizer compartido)
curl https://auth-api-dev.paqta.app/api/v1/auth/me

# Benchmarks (mismo benchmark.js entregado por la cátedra)
node benchmark.js https://magic-links-api-dev.paqta.app/api/v1/magic-links/<token>/public 200
node benchmark.js https://auth-api-dev.paqta.app/api/v1/auth/me 200
```

## 🏗️ Arquitectura real

```mermaid
flowchart LR
    FE[Frontend Next.js paqta] -->|HTTPS| AUTH[pq-auth-ms]
    FE -->|HTTPS| AGENCY[pq-agency-ms]
    FE -->|HTTPS| BILLING[pq-billing-ms]
    FE -->|HTTPS| FILES[pq-files-ms]
    FE -->|HTTPS| PROP[pq-proposals-ms]
    FE -->|HTTPS| MAGIC[pq-magic-link-ms]

    subgraph "Único camino síncrono multi-salto real: DELETE /api/v1/auth/account"
        AUTH -->|fetch await, Promise.allSettled| BILLING
        AUTH -->|fetch await, Promise.allSettled| PROP
        AUTH -->|fetch await, Promise.allSettled| FILES
        AUTH -->|fetch await, Promise.allSettled| MAGIC
    end

    subgraph "Acoplamiento oculto (no es un camino de datos, es infraestructura compartida)"
        SSM[(SSM: /paqta/{stage}/auth/authorizer-arn)]
        AGENCY -.->|lee ARN al desplegar| SSM
        BILLING -.->|lee ARN al desplegar| SSM
        FILES -.->|lee ARN al desplegar| SSM
        PROP -.->|lee ARN al desplegar| SSM
        MAGIC -.->|lee ARN al desplegar| SSM
        AUTH -.->|publica ARN| SSM
    end
```

- Topología principal: **estrella** — el frontend llama a cada microservicio de forma independiente,
  no hay un Gateway interno que reencadene llamadas (`paqta/lib/api/config.ts:2-7`).
- El **único** camino síncrono con múltiples saltos reales del backend es el borrado de cuenta:
  `pq-auth-ms/handlers/delete-account.ts` llama por HTTPS a billing/proposals/files/magic-links
  **en paralelo**, esperando cada respuesta (`fetch` + `await`), pero tolerando fallos individuales.
- No existe ningún camino asíncrono por eventos (SQS/SNS/EventBridge/Redis) en el sistema real.

## 🗺️ Patrones y principios aplicados (evidencia real)
- **Lambda Authorizer compartido (Sentinel/Guard distribuido):** `pq-auth-ms/sst.config.ts:41-46` publica
  el ARN del `ClerkAuthorizer` en SSM (`/paqta/{stage}/auth/authorizer-arn`); cada servicio lo importa
  (`pq-agency-ms/sst.config.ts:33-36,54`) y protege sus rutas (`auth: { lambda: lambdaAuthorizer.id }`).
  Es el patrón más cercano a un "Gateway" del sistema, aunque distribuido en vez de centralizado.
- **Resiliencia ante fallos parciales (Promise.allSettled):** `delete-account.ts:132-133` no usa
  `Promise.all` (que fallaría entero si un downstream cae), sino `allSettled`, capturando el resultado
  de cada purga por separado (`delete-account.ts:165-211`) y devolviendo `207 Multi-Status` (línea 216).
- **Manejo de excepciones:** `try/catch` de nivel superior en el handler (`delete-account.ts:76-230`) con
  respuesta de error uniforme `errorResponse(event, 500, "INTERNAL_ERROR", ...)` (línea 229); errores por
  sub-operación capturados independientemente sin abortar el resto del flujo.
- **DIP / SRP por archivo:** cada handler es una función pura exportada (sin clases/controladores), con
  helpers compartidos `response()`/`errorResponse()` (`pq-auth-ms/utils.ts:12,22`) reutilizados en todos
  los repos — separación de responsabilidades por convención de carpeta (`handlers/` vs `utils.ts`) en vez
  de inyección de dependencias con contenedor.
- **Anti-patrón detectado (acoplamiento de datos):** `pq-proposals-ms/billing-client.ts:1-56` **no llama a
  la API** de `pq-billing-ms`; lee y escribe directamente su tabla DynamoDB `BillingUsage` mediante el SDK.
  Es acoplamiento fuerte a nivel de esquema de datos, más difícil de versionar que una llamada HTTP.

## 🧭 Metodología
- **Kanban:** tabla Markdown en [`TABLERO_KANBAN.md`](./TABLERO_KANBAN.md) (no se usó GitHub Projects este corte).
- **Ramificación:** este hub usa **GitHub Flow** (`main` + ramas `docs/...`, PR simple, tag por avance). Los
  6 repos de producto (`pq-*-ms`) mantienen su propio historial y ramificación previos al proyecto integrador;
  no se modificó ningún commit existente para esta tarea.
- **Commits semánticos:** Conventional Commits. Ejemplos reales de este avance:
  ```
  docs(readme): documentar arquitectura real de paqta (sin simular TCP/eventos)
  docs(bench): agregar mediciones reales contra magic-links-api-dev y auth-api-dev
  docs(kanban): actualizar tablero con evidencia del Avance 1
  chore(gitignore): excluir repos hermanos y scripts operativos del hub de docs
  ```

---

## 🟢 Avance 1 — Acoplamiento temporal y latencia · `tag v1-avance1`

### Qué existe realmente vs. qué pide la Tarea 1
| Requerido | ¿Existe en paqta real? | Evidencia |
|---|---|---|
| Gateway + 3 microservicios | Parcial — 6 microservicios reales, sin Gateway interno (frontend llama directo) | `paqta/lib/api/config.ts:2-7` |
| Camino síncrono TCP, ≥2 saltos | Sí en espíritu (HTTPS, no TCP crudo): `pq-auth-ms` → 4 servicios en paralelo | `delete-account.ts:121-133` |
| Camino asíncrono por eventos (Redis/cola) | **No existe** — cero SQS/SNS/EventBridge/Redis en los 6 repos | búsqueda exhaustiva sin resultados |
| Manejo de excepciones en capa de servicio | Sí | `delete-account.ts:76-230`, `billing-ms/dodo-client.ts:29` |

### 📈 Latencia real (endpoints en vivo, stage `dev`, 200 peticiones c/u)
| Endpoint | Descripción | Promedio (ms) | p95 (ms) | Máx (ms) |
|---|---|---|---|---|
| `GET /api/v1/magic-links/{token}/public` (pq-magic-link-ms) | Ruta **pública**, sin autorizador, 1 salto (API Gateway → Lambda → DynamoDB) | 138.59 | 157.00 | 503.00 |
| `GET /api/v1/auth/me` (pq-auth-ms) | Ruta **protegida**, pasa primero por el Lambda Authorizer compartido antes del handler | 130.97 | 147.00 | **1038.00** |

> Ambas peticiones se responden con error esperado (404 "token no encontrado" / 401 "sin sesión") — no
> se ejecutó ninguna operación destructiva. Lo que se mide es la **latencia real de infraestructura**
> (API Gateway + Lambda, con cold starts), no el resultado de negocio. Salidas completas en
> [`docs/bench-real-sincrono.txt`](./docs/bench-real-sincrono.txt) y
> [`docs/bench-real-con-authorizer.txt`](./docs/bench-real-con-authorizer.txt).

### 🧨 Acoplamiento temporal — lo que se puede evidenciar sin tocar producción
No se apagó ningún servicio real (no hay proceso que "apagar": son funciones Lambda; forzar una caída
implicaría desplegar código roto sobre infraestructura compartida, algo que se descartó por seguridad).
En su lugar, el acoplamiento se evidencia **por lectura de código + comportamiento observado**:

1. **Máximo de 1038 ms en la ruta protegida** vs 503 ms en la pública: el salto adicional obligatorio del
   Lambda Authorizer (que valida el JWT de Clerk de forma síncrona antes de invocar el handler) es un
   punto de latencia y de acoplamiento compartido por **los 6 microservicios**: si `pq-auth-ms` deja de
   poder desplegar/actualizar su autorizador, ningún servicio protegido puede autenticar peticiones,
   aunque sus propios handlers y bases de datos estén sanos (`pq-agency-ms/sst.config.ts:33-36,54`).
2. **`delete-account.ts:132-133` usa `Promise.allSettled`, no `Promise.all`**: si en el momento del borrado
   `billing-api-dev` estuviera caído, la promesa de esa purga específica se rechaza, se captura en el
   `catch` de su propio bloque (`delete-account.ts:165-171` para billing) y el resto de purgas continúa;
   la respuesta final es `207` con el detalle de qué servicio falló. Esto es lo opuesto al acoplamiento
   temporal "todo o nada" que describe la guía: el sistema real ya mitiga el efecto cascada, a costa de
   dejar datos parcialmente sin purgar en el servicio caído (un problema distinto: **consistencia**, no
   disponibilidad).

### 🧠 Análisis
En el camino síncrono clásico que describe la Tarea 1 (Gateway → A → B esperando respuesta), la latencia
se acumula porque cada salto es bloqueante: el llamador no puede responder hasta que el llamado responde.
En paqta real esto se observa de forma indirecta: la ruta protegida (`/auth/me`) siempre paga el costo
adicional del Lambda Authorizer antes de ejecutar su propio handler — dos invocaciones Lambda síncronas
en cadena por petición, lo que explica su cola de latencia más larga (máx. 1038 ms vs 503 ms). El
**acoplamiento temporal**, en el sentido estricto de la guía (si un servicio de la cadena cae, toda la
petición falla), **no se manifiesta como un "todo o nada"** en el único camino real con múltiples saltos
(`delete-account.ts`), porque el equipo de paqta ya adoptó `Promise.allSettled` en vez de `Promise.all`:
es una decisión de diseño consciente para evitar exactamente el problema que la Tarea 1 pide demostrar.
Donde sí existe un acoplamiento temporal genuino y no mitigado es en el **Lambda Authorizer compartido**:
es una dependencia dura, en cada petición, de los 6 microservicios hacia la disponibilidad de ese único
recurso (y transitivamente, hacia Clerk como proveedor externo). La razón de fondo por la que **no hay
camino asíncrono por eventos** en el sistema real es arquitectónica: al ser serverless con DynamoDB,
el equipo resolvió el desacople de otra forma —acceso directo a tablas compartidas
(`pq-proposals-ms/billing-client.ts:1-56`)— que desacopla en el tiempo de ejecución pero acopla en el
esquema de datos, un trade-off distinto al que Redis/eventos habría resuelto.

---

## 🔵 Nota sobre alcance y rúbrica
Por decisión explícita del equipo, este avance **documenta la arquitectura real de paqta tal como existe**,
sin construir infraestructura nueva de TCP/Redis para simular artificialmente lo que pide la guía. Esto
implica que el criterio **C1 (3 MS + Gateway con TCP y Redis)** y parte de **C2 (tabla de ambos caminos +
prueba de caída clásica)** no se cumplen en su forma literal, porque paqta real no tiene esos transportes.
Sí se cumple: 3+ microservicios reales con Gateway-equivalente (autorizador compartido), medición real de
latencia con evidencia verificable, y un análisis de acoplamiento temporal basado en comportamiento real
de producción en vez de una demo sintética.
