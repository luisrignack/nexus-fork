# Funcionalidades tipo Pro / Pro+ — lista priorizada

> Este documento convierte las capacidades que Nexus Pro/Pro+ ofrece en binario propietario en una
> lista de **reimplementaciones limpias** sobre el fork EPL-1.0.
>
> Regla legal: **no se porta ni se descompila código propietario**. Cada función se diseña desde
> especificaciones públicas, RFCs y el comportamiento observable de la API. Nada de copiar clases,
> recursos ni cadenas de las ediciones comerciales.

## 1. Punto de partida del fork

Ya implementado y verificado en el fork:

| Área | Estado |
|---|---|
| Formatos | npm (hosted/proxy/group + search/audit/rebuild), Yum (hosted/proxy), PyPI (hosted/proxy/group), Helm, Go, RubyGems (los tres con validación E2E ✅ 2026-10-24), Docker/OCI (proxy pull-through) |
| Docker en UI/API | `docker-proxy` es un formato real: aparece en `/service/rest/internal/ui/repositories/recipes` y se crea por REST/UI (✅ 2026-10-24) |
| Datastore | H2 embebido y PostgreSQL externo |
| Blob store | File (incluido en OSS); S3 presente en el árbol público |
| Núcleo | REST v1/beta, RBAC, tareas, browse/search, UI CoreUI |
| Identidad | **SSO OIDC** (✅ 2026-10-24, `51add5a`), **User Tokens** (✅ `eebed1c`/`454338a`), **LDAP** (realm + configuración + pantalla de administración, ✅ 2026-10-24, `fa46a5e` + `2df79cf`) |
| Operación | **GC de la caché Docker** (✅ `5552712`), **SBOM CycloneDX/SPDX** (✅ `8913d17`), **backup/restore con manifiesto** (✅ `fe3e0e4`) |

Lo que falta para acercarse a Pro/Pro+ es, sobre todo, **operación a escala** (HA, DR, SSO),
**gobierno de contenido** (staging, cuarentena, licencias) y **cumplimiento** (FIPS, auditoría).

## 1.1. Hallazgo: varias pantallas Pro ya están en el árbol

El `nexus-coreui-plugin` público incluye componentes React de funciones Pro, pero sin backend y
ocultos por el *gating* de edición/licencia. No son código Pro copiado: son pantallas EPL que
quedaron en el árbol y hoy no tienen API que las alimente.

| Pantalla | Ruta UI | Estado real | Qué falta |
|---|---|---|---|
| User Tokens | `/admin/security/usertoken` | ✅ **Completado (2026-10-24, `eebed1c`)** — UI + backend REST + realm + capability | — |
| SAML | `/admin/security/saml` | UI de configuración | Realm Shiro SAML + metadatos SP/ACS |
| OAuth2 / OIDC | `/admin/security/oauth2` | ✅ **Completado (2026-10-24)** — UI + realm + callback + capability | — |
| Atlassian Crowd | `/admin/security/atlassiancrowd` | UI de configuración | Cliente Crowd + realm |
| Nodes / HA | `/admin/system/nodes` | UI de estado | Modelo de nodos, heartbeat y coordinación |
| Licensing | `/admin/system/licensing` | UI de instalación de licencia | Motor de licencias propio (o eliminar la pantalla) |
| Malware Remediation | vista de browse | UI de cuarentena | Motor de escaneo (ver 4.6) |
| **LDAP** | `/admin/security/ldap` | ✅ **Completado (2026-10-24)** — pantalla React + realm + capability + REST | — |

Para habilitarlas hay que hacer dos cosas: implementar el backend y ajustar
`visibilityRequirements` en `routerConfig/routes/*.js` (quitar `editions: ['PRO']` y
`licenseValid`, sustituyéndolos por permisos reales). **User Tokens es el caso más rentable**:
la UI y los permisos ya existen y solo falta el endpoint.

## 2. Priorización resumida

Escala de esfuerzo: **S** ≤ 2 semanas, **M** 3-6 semanas, **L** 2-4 meses, **XL** > 4 meses
(una persona; se reduce con paralelización). Riesgo: probabilidad de romper compatibilidad,
rendimiento o integridad de datos.

| # | Función | Valor | Esfuerzo | Riesgo | Fase |
|---|---|---|---|---|---|
| 1 | SSO OIDC ~~(y después SAML)~~ ✅ hecho (2026-10-24) — queda SAML | Alto | M | Bajo | P0 |
| 2 | ~~UI de administración del proxy Docker~~ ✅ hecho (2026-10-24) | Medio | S | Bajo | P0 |
| 3 | ~~GC y tareas de mantenimiento de la caché Docker~~ ✅ hecho (2026-10-24, `5552712`) | Medio | S | Bajo | P0 |
| 4 | ~~Exportación SBOM (CycloneDX/SPDX)~~ ✅ hecho (2026-10-24, `8913d17`) | Alto | M | Bajo | P0 |
| 5 | ~~Backup/restore consistente (DB + blobs)~~ ✅ hecho (2026-10-24, `fe3e0e4`) | Alto | M | Medio | P0 |
| 6 | Soft-delete + papelera/restore | Alto | M | Medio | P1 |
| 7 | Staging y promoción de artefactos | Alto | L | Medio | P1 |
| 8 | Replicación de contenido / DR activo-pasivo | Alto | L | Alto | P1 |
| 9 | Blob stores Azure/GCS (S3 ya existe) | Medio | M | Bajo | P1 |
| 10 | SCIM 2.0 para usuarios/grupos | Medio | M | Bajo | P1 |
| 11 | Escaneo de malware + cuarentena | Alto | L | Alto | P1 |
| 12 | Políticas de licencia y bloqueo de componentes | Medio | L | Medio | P2 |
| 13 | HA activo-activo (clustering) | Muy alto | XL | Muy alto | P2 |
| 14 | FIPS 140-3 | Alto (regulado) | XL | Muy alto | P2 |
| 15 | Gobierno de dependencias AI/MCP | Emergente | M | Medio | P2 |
| 16 | Auditoría inmutable y reportes de cumplimiento | Medio | M | Bajo | P2 |

## 3. Fichas de las funciones P0

### 3.1. SSO con OIDC ✅ COMPLETADO (2026-10-24) — SAML pendiente

**Estado OIDC.** Implementado como realm `OAuth2Realm` (`OidcRealm` en `nexus-security`) más la
lógica de flujo y persistencia en `nexus-core` (`org.sonatype.nexus.internal.security.oidc`):

- **Flujo de navegador:** `GET /oidc/login` (servlet en la raíz, contrato de la UI `SsoLogin.jsx`)
  genera `state`/`nonce` de un solo uso (caché Guava con TTL de 5 min) y redirige al
  authorization endpoint del IdP; `GET /oidc/callback` canjea el `code` por `id_token`, consume el
  `state` (no reutilizable) y delega en el realm vía `subject.login(...)`.
- **Validación del `id_token`** (`OidcService`, con `java-jwt` 3.16.0): firma contra JWKS (URL o
  inline), `iss` (con `exactMatchClaims` para issuer multi-tenant), `aud` = `clientId`, `exp`/`nbf`
  y `nonce`.
- **Provisioning JIT:** usuarios desconocidos se crean en `UserManager.DEFAULT_SOURCE` (activo, con
  nombre/email según claims configurables); usuarios existentes actualizan solo los atributos
  presentes en el token.
- **Mapeo de grupos:** el claim configurado (`groupsClaim`, lista o string) se cruza con los IDs de
  rol existentes; los mapeos se guardan bajo `OAuth2Realm` con `setUsersRoles`.
- **Persistencia:** capability `oidc` creada deshabilitada por `OidcCapabilityBooter`; la config se
  guarda en `OidcConfigurationStoreImpl` sobre `CapabilityRegistry`, con el client secret cifrado
  vía `SecretsService` (placeholder en la UI para no sobrescribirlo) y privilegios
  `nx-oidc-settings-read/update`.
- **UI:** ruta `/admin/security/oauth2` des-gated (solo `Permissions.OIDC_SETTINGS.READ`), estado
  `oauth2Enabled` publicado por `OidcStateContributor` para que la página de login muestre el botón
  SSO.
- **Tests:** `OidcServiceTest` (8), `OidcConfigurationStoreImplTest` (7), `OidcRealmTest` (6),
  `NexusSecurityContributorTest` (2); suites de `nexus-security` (362), `nexus-base` (151) y
  `nexus-core` (368) en verde.

**SAML (pendiente).** Realm Shiro SAML + metadatos SP/ACS reutilizando el modelo de mapeo de
grupos; librería `opensaml` (versión compatible EPL/APL) en `dependencyManagement`.

**Validación con IdPs reales.** Probado end-to-end con un IdP de prueba; queda validar con
Keycloak y Azure Entra ID.

### 3.2. UI de administración del proxy Docker — ✅ COMPLETADO (2026-10-24)

**Estado.** Implementado como formato de primera clase `docker` + receta `docker-proxy`, no como
pantalla aparte. Consecuencias:

- El formato aparece en `GET /service/rest/internal/ui/repositories/recipes` y en el asistente de
  creación de repositorios de la UI.
- `POST /service/rest/v1/repositories/docker/proxy` crea un repositorio Docker (201).
- El enrutado es por ruta: `docker pull <host>/<repo>/<imagen>`; el primer segmento tras `/v2/`
  selecciona el repositorio. Sin coincidencia se usa el selector global de upstreams
  (`dockerhub`, `gcr`, `ghcr`, `quay`).
- El formulario expone `dockerProxy.indexUrl`, `indexType`, foreign layers y credenciales
  Basic/Bearer (cifradas con `SecretsService`).
- Stores de contenido propios (`DockerComponentDAO`, `DockerAssetDAO`, `DockerAssetBlobDAO`,
  `DockerBrowseNodeDAO`) para que browse/search y las tareas de mantenimiento funcionen.

**Lo que queda de esta ficha (opcional).** Pantalla dedicada de estado de caché (tamaño, nº de
entradas, purga). Hoy se cubre con la tarea de GC del punto 3.3.

**Verificación.** `docker pull localhost:8081/docker-ghcr/astral-sh/uv:latest` completo; 29 tests
del módulo Docker en verde; commit `3c8fab9`.

### 3.3. GC de la caché Docker — ✅ COMPLETADO (2026-10-24, `5552712`)

**Implementado.** Tarea programada `repository.docker-gc` con límite de tamaño por upstream
(LRU sobre `cachedAt` de los `.meta`), edad máxima opcional, modo *dry-run* con reporte y
exclusión de blobs referenciados por manifests recientes.

**Verificación.** Tests del módulo en verde; ejecución documentada en `ESTADO_PROYECTO.md`.

### 3.4. Exportación SBOM — ✅ COMPLETADO (2026-10-24, `8913d17`)

**Implementado.** Endpoint REST de exportación de inventario en **CycloneDX 1.5** y **SPDX 2.3**
(módulo `nexus-sbom`), con purl, hashes y relaciones; recorre componentes y assets por repositorio.

**Pendiente opcional.** Filtros por fecha, salida XML y relaciones de dependencia enriquecidas
desde POM/`package.json`/`requires_dist`.

### 3.5. Backup/restore consistente — ✅ COMPLETADO (2026-10-24, `fe3e0e4`)

**Implementado.** Tarea de backup de blobs y datastore con manifiesto verificado (versión y
checksums), módulo `nexus-backup` con `BackupFileManager`.

**Pendiente opcional.** Automatizar el restore en instancia limpia como tarea y validar la
ventana de mantenimiento en el despliegue con PostgreSQL.

## 4. Fichas P1

### 4.1. Soft-delete + papelera

Guardar componentes/assets borrados en tablas de papelera con retención configurable y restaurarlos
por API/UI. Alternativa más barata: *tombstones* + blob retention. **Esfuerzo M, riesgo medio**
(hay que ajustar todos los DAOs de borrado y la búsqueda para excluir la papelera).

### 4.2. Staging y promoción

Repositorios de staging separados de release, reglas de promoción (aprobación, tests, ventana),
copia atómica entre repos y trazabilidad. **Esfuerzo L, riesgo medio**; depende de un modelo de
metadatos de promoción y de permisos nuevos.

### 4.3. Replicación de contenido / DR

Replicación activo-pasivo entre dos instancias: export incremental por componente + blobs,
resolución de conflictos y *cutover* documentado. **Esfuerzo L, riesgo alto**; requiere idempotencia
y verificación de hashes. No confundir con HA activo-activo.

### 4.4. Blob stores Azure/GCS

S3 ya está en el árbol público. Implementar `BlobStore` para Azure Blob y Google Cloud Storage
reutilizando la interfaz y los tests de S3. **Esfuerzo M, riesgo bajo.**

### 4.5. SCIM 2.0

Endpoints `/scim/v2/Users` y `/Groups` para aprovisionar desde Okta/Entra. **Esfuerzo M, riesgo
bajo**; encaja con el trabajo de SSO.

### 4.6. Escaneo de malware + cuarentena

Integrar ClamAV (ICAP o proceso local) o un motor externo; cuarentena de assets sospechosos,
bloqueo de descarga hasta revisión, cuarentena automática y liberación manual. **Esfuerzo L, riesgo
alto** (falsos positivos, latencia de subida, aislamiento de blobs).

## 5. Fichas P2

### 5.1. HA activo-activo (clustering)

Requisitos: blob store compartido, sesiones compartidas, tareas con bloqueo distribuido, cachés
coherentes, elección de líder para tareas singleton, *health checks* y *draining*. **Esfuerzo XL,
riesgo muy alto.** Recomendación: no empezar hasta tener PostgreSQL, backup/restore y DR estables.
Una HA activo-pasivo (dos nodos, un líder, failover manual/automático) es un paso intermedio mucho
más realista.

### 5.2. FIPS 140-3

Requiere JCE en modo FIPS, proveedor validado (BouncyCastle FIPS), desactivar algoritmos no
permitidos, auditoría de TLS y de almacenamiento de secretos, y un build reproducible. **Esfuerzo
XL, riesgo muy alto**; normalmente exige certificación externa, no solo código.

### 5.3. Políticas de licencia

Motor de reglas sobre licencias declaradas/detectadas (allow/deny/warn), bloqueo en subida y
reporte de cumplimiento. Depende de SBOM y de una base de licencias. **Esfuerzo L, riesgo medio.**

### 5.4. Gobierno de dependencias AI/MCP

Proxy/registro para modelos, *prompts*, servidores MCP y artefactos de IA; políticas de procedencia,
escaneo y versionado. El ecosistema aún se está estandarizando. **Esfuerzo M, riesgo medio**;
empezar por un repositorio `raw`/OCI para artefactos MCP y un catálogo con metadatos.

### 5.5. Auditoría inmutable y reportes

Exportación firmada/hash-chained del audit log, retención, búsqueda y reportes (accesos, cambios,
borrados). **Esfuerzo M, riesgo bajo.**

## 6. Ruta recomendada

| Fase | Contenido | Objetivo |
|---|---|---|
| **Fase A (P0)** | ~~OIDC~~ ✅, ~~UI/config Docker~~ ✅, ~~GC Docker~~ ✅, ~~SBOM~~ ✅, ~~backup/restore~~ ✅ — queda **SAML** | Operación diaria y cumplimiento básico sin tocar el núcleo |
| **Fase B (P1)** | Soft-delete, staging, SCIM, Azure/GCS, DR, malware | Gobierno de contenido y resiliencia |
| **Fase C (P2)** | HA activo-pasivo → activo-activo, FIPS, licencias, AI/MCP | Escala y requisitos regulados |

## 7. Reglas de implementación

1. Cada función entra como **módulo/plugin** con su `pom.xml`, tests y documentación propia.
2. No modificar el comportamiento por defecto de los formatos existentes salvo *feature flag*
   explícito.
3. Toda configuración nueva debe poder fijarse por propiedad/variable de entorno (como Docker y
   PostgreSQL), para automatizar despliegues.
4. Añadir tests unitarios y al menos una prueba manual documentada por función.
5. Actualizar `ESTADO_PROYECTO.md` y la documentación de `docs/` al cerrar cada función.

---

## 8. Informe: qué falta por incorporar (actualizado 2026-10-24)

> Fuente: el propio árbol público del fork (rutas con `editions: ['PRO']`, `bundle: 'nexus-*-plugin'`
> inexistentes y módulos presentes/ausentes) más el catálogo de las secciones 1–7. No se ha usado
> ni se va a usar código de las ediciones comerciales.

### 8.1. Ya incorporado (Fase A completada salvo SAML)

| Función | Commit | Verificación |
|---|---|---|
| SSO OIDC/OAuth2 (realm, flujo, UI, capability, provisioning JIT) | `51add5a` | `OidcServiceTest` 8, `OidcConfigurationStoreImplTest` 7, `OidcRealmTest` 6; suites `nexus-security` 362 / `nexus-base` 151 / `nexus-core` 368 en verde |
| UI/API del proxy Docker (`docker-proxy` como formato real) | `3c8fab9`, `d21da80` | `docker pull localhost:8081/docker-ghcr/...` completo |
| GC de la caché Docker (`repository.docker-gc`) | `5552712` | tests del módulo en verde |
| SBOM CycloneDX 1.5 / SPDX 2.3 (`nexus-sbom`) | `8913d17` | endpoint REST de exportación |
| Backup de blobs + datastore con manifiesto verificado (`nexus-backup`) | `fe3e0e4` | manifiesto con checksums |
| User Tokens (backend, realm, REST, privilegios) | `eebed1c` / `454338a` | 30 tests en `nexus-core` |
| LDAP: realm, configuración, REST de administración | `fa46a5e` | 17 ficheros, +3.024 líneas |
| LDAP: pantalla de administración en la UI React | `2df79cf` | 43 tests React LDAP; `readTemplates`/`verify*`/`clearCache` en verde |
| Formatos Helm, Go, RubyGems validados E2E | `e4055d9` | `docs/VALIDACION_HELM_GO_RUBYGEMS.md` |

### 8.2. Fase A — lo que queda (P0)

| # | Qué | Esfuerzo | Riesgo | Notas |
|---|---|---|---|---|
| 1 | **SAML 2.0** | M | Medio | Último SSO. La ruta `/admin/security/saml` sigue tras `bundle: 'nexus-saml-plugin'` (inexistente) + `editions: ['PRO']`. Requiere realm Shiro SAML, metadatos SP/ACS y librería SAML con licencia compatible. Alternativa más barata: **SAML a través del proxy OIDC de Keycloak/Entra**, sin código propio. |
| 2 | **Formateo/refs del SBOM** | S | Bajo | Salida XML, filtros por fecha/repo y relaciones de dependencia desde POM/`package.json`/`requires_dist`. |
| 3 | **Restore automático del backup** | S–M | Medio | Hoy el backup está; falta la tarea de restore + validación de integridad en instancia limpia. |
| 4 | **Pantalla de estado de caché Docker** | S | Bajo | Tamaño, nº de entradas y purga manual; el GC ya hace la parte de borrado. |
| 5 | **SAML/Crowd**: evaluar si merece la pena frente a OIDC | — | — | `/admin/security/atlassiancrowd` sigue tras `bundle: 'nexus-crowd-plugin'` + `licenseValid`. |

### 8.3. Fase B — P1 (gobierno de contenido y resiliencia)

| # | Función | Esfuerzo | Riesgo | Punto de partida en el árbol |
|---|---|---|---|---|
| 6 | **Tag management / etiquetado de componentes** | M | Bajo | La ruta `/browse/tags` ya existe en `browseRoutes.js` con `editions: ['PRO']`; faltan backend, DAO y REST. |
| 7 | **Soft-delete + papelera/restore** | M | Medio | Toca el modelo de componentes y el índice de búsqueda. |
| 8 | **Staging y promoción** | L | Medio | Requiere estados por componente y políticas de transición. |
| 9 | **Blob stores Azure/GCS** | M | Bajo | Existe el patrón en `nexus-blobstore-s3` (adaptador `BlobStore` + UI); se replica para Azure y GCS. |
| 10 | **SCIM 2.0 para usuarios/grupos** | M | Bajo | Complemento natural de OIDC (ya con provisioning JIT). |
| 11 | **Replicación de contenido / DR activo-pasivo** | L | Alto | El core ya tiene `nexus-distributed-event-service-api`. |
| 12 | **Escaneo de malware + cuarentena** | L | Alto | La vista `/browse/malwarerisk` existe (`statesEnabled: MALWARE_RISK_ENABLED`, `editions: ['PRO']`); falta motor de escaneo (ClamAV u otro) y flujo de cuarentena. |
| 13 | **RBAC por repositorio/upstream en el servlet Docker** | S | Bajo | Ver `docs/DOCKER.md` §5. |

### 8.4. Fase C — P2 (escala y cumplimiento)

| # | Función | Esfuerzo | Riesgo | Notas |
|---|---|---|---|---|
| 14 | **HA activo-activo (clustering)** | XL | Muy alto | La ruta `/admin/system/nodes` está des-gated; el modelo de nodos/heartbeat no existe. |
| 15 | **FIPS 140-3** | XL | Muy alto | Criptografía certificada: depende de proveedor Bouncy Castle FIPS y de la JDK. |
| 16 | **Políticas de licencia y bloqueo de componentes** | L | Medio | Con SBOM ya hay inventario; falta el motor de políticas. |
| 17 | **Auditoría inmutable y reportes de cumplimiento** | M | Bajo | Hay `nexus-audit` y `nexus-webhooks`; falta inmutabilidad (WORM/object lock) y reportes. |
| 18 | **Gobierno de dependencias AI/MCP** | M | Medio | Formato nuevo + políticas; terreno sin explorar en el ecosistema. |

### 8.5. Formatos y stores aún ausentes

- **Formatos no incluidos:** NuGet, Conan, Cargo, CocoaPods, p2/OSGi, Swift, Git LFS. Todos siguen
  el mismo patrón que los ya hechos (`formats/nexus-repository-<fmt>` + recipe + stores + REST):
  **NuGet** es el candidato natural por ser el análogo de npm (feed v3 + `nuspec`).
- **Stores:** solo `file` y `s3` (Azure/GCS → ver 8.3 #9).
- **Datastores:** H2 + PostgreSQL ya funcionan; la ruta `/admin/repository/datastore` sigue pidiendo
  `bundle: 'nexus-pro-datastore-plugin'` — conviene des-gatearla para gestionar la DB desde la UI.

### 8.6. Lo que NO se debe portar (y por qué)

Funcionalidades propietarias de las ediciones comerciales: **Firewall / Inspector / Advanced
Security**, escaneo de vulnerabilidades gestionado por Sonatype, licencias comerciales, y el
catálogo de telemetría/soporte (`/admin/support/supportrequest`, `Permissions.ATLAS`). Son código y
servicios cerrados; portarlos violaría la licencia. Sí es legítimo **integrarlos por API** si se
contrata el servicio, y cubrir el caso de uso con soluciones abiertas (p. ej. escaneo con Trivy/Grype
+ políticas propias para sustituir el escaneo gestionado).

### 8.7. Orden recomendado tras cerrar la Fase A

1. **SAML** (o delegarlo al proxy OIDC) → cierra identidad.
2. **Tag management** → pantalla ya montada, retorno rápido.
3. **NuGet** → formato de mayor demanda pendiente.
4. **Azure/GCS** → sigue el patrón de S3.
5. **Soft-delete + papelera** → base de staging/DR.
6. **SCIM** → identidad a escala.
7. **Malware/auditoría/políticas** → cumplimiento, una vez haya inventario y etiquetas.
8. **HA / FIPS** → solo si hay requisito explícito; coste alto.
