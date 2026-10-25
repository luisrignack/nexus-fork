# Diseño — Plugin de formato npm para Nexus Repository Core

> Documento de diseño. Complementa a `ESTADO_PROYECTO.md`.
> Objetivo: añadir soporte del registro npm al fork, empezando por un MVP funcional y verificable.

## 1. Alcance del MVP

Estado: **F1 (hosted), F2 (proxy), F3 (group) y F4 (extras) implementados y verificados**
(2026-10-23).

Un repositorio npm **hosted** que permita:

- `npm publish` de un paquete (incluye `package.json` + tarball).
- `npm install` / `npm view` (packument y tarball).
- `npm dist-tag add|rm|ls`.
- Autenticación con las credenciales de Nexus (Basic auth).

Además, el proxy cachea paquetes de `registry.npmjs.org` y el group agrega hosted+proxy con
merge de versiones y dist-tags.

F4 completado: `npm audit`, búsqueda `/-/v1/search`, tarea de rebuild de metadatos y limpieza del
formulario de receta. Quedan para fases posteriores los tokens granulares y las features tipo Pro
(ver `docs/FUNCIONALIDADES_PRO.md`).

## 2. Protocolo npm (endpoints implementados)

| Método | Ruta | Uso |
|---|---|---|
| GET | `/{package}` | Packument completo (todas las versiones, dist-tags) |
| GET | `/{package}/{version}` | Manifiesto de una versión |
| GET | `/{package}/-/{tarball}` | Descarga del `.tgz` |
| PUT | `/{package}` | Publicar (body con `_attachments`, base64 del tgz) |
| DELETE | `/{package}/-rev/{rev}` | Unpublish (lo usa `npm unpublish --force`) |
| PUT | `/-/package/{package}/dist-tags/{tag}` | Crear/actualizar dist-tag |
| DELETE | `/-/package/{package}/dist-tags/{tag}` | Borrar dist-tag |
| GET | `/-/package/{package}/dist-tags` | Listar dist-tags |
| GET | `/-/whoami` | Usuario autenticado (lo usan algunas CLIs) |
| GET | `/-/ping` | Comprobación de disponibilidad |
| PUT | `/-/user/org.couchdb.user:{name}` | Login legacy de npm (opcional, no implementado) |

Nombres con scope (`@scope/name`) llegan URL-encoded como `%40scope%2Fname`.

Content types: `application/json` para metadatos, `application/octet-stream` para tarballs.
El tarball publicado llega en `_attachments["<name>-<version>.tgz"].data` (base64) y debe
guardarse con checksum `sha1` (y `sha512` si viene en `dist.integrity`).

## 3. Mapeo a conceptos de Nexus

| npm | Nexus |
|---|---|
| Paquete (con scope) | `Component` con `name` = `@scope/nombre` o `nombre`, `group` = scope |
| Versión | `Component.version` + atributos (`dist-tags`, `deprecated`) |
| `package.json` de cada versión | `Asset` JSON en `/{paquete}/{version}` |
| Tarball `.tgz` | `Asset` binario (blob) en `/{paquete}/-/{fichero}.tgz` |
| `dist-tags` | Atributo de componente `dist-tags`; `latest` se deriva de la versión estable más alta |

Rutas de almacenamiento usadas (compatibles con el layout npm):
`@scope/nombre/{version}` para el manifiesto y `@scope/nombre/-/nombre-version.tgz` para el tarball.
El packument completo no se almacena como asset en el flujo nuevo: se genera dinámicamente desde
componentes y manifiestos. Se conserva la lectura del packument legacy en `/{paquete}` como fallback.

## 4. Clases implementadas (siguiendo el plugin `raw`)

Referencia: `public/common/components/formats/nexus-repository-raw/`.

```
public/common/components/formats/nexus-repository-npm/
├── src/main/java/org/sonatype/nexus/
│   ├── bootstrap/npm/NpmConfiguration.java
│   ├── content/npm/
│   │   ├── NpmContentFacet.java
│   │   └── internal/
│   │       ├── NpmContentFacetImpl.java
│   │       ├── NpmDistTagsManager.java
│   │       ├── browse/NpmBrowseNodeDAO.java
│   │       ├── store/  (NpmAssetDAO, NpmComponentDAO, NpmContentRepositoryDAO,
│   │       │            NpmAssetBlobDAO, NpmStoresProvider)
│   │       └── recipe/
│   │           ├── NpmRecipeSupport.java
│   │           ├── NpmHostedRecipe.java        # F1
│   │           ├── NpmProxyRecipe.java         # F2
│   │           ├── NpmGroupRecipe.java         # F3
│   │           ├── NpmContentHandler.java      # GET/PUT/DELETE hosted
│   │           ├── NpmPackumentBuilder.java    # packument dinámico
│   │           ├── NpmProxyFacet.java          # F2
│   │           ├── NpmProxyHandler.java        # F2, reescritura dist.tarball
│   │           └── NpmGroupHandler.java        # F3, merge de packuments/dist-tags
│   ├── repository/npm/
│   │   ├── internal/ (NpmFormat, NpmSecurityFacet, contribuidor de seguridad,
│   │   │              NpmCoordinatesHelper, NpmFormatSecurityContributor)
│   │   └── rest/ (hosted, proxy y group: requests, converters, API models,
│   │              resources V1/Beta, NpmRepositoryAdapter)
│   └── ...
└── src/test/java/.../NpmCoordinatesHelperTest.java
```

Puntos de extensión confirmados en el código:

- `Format`: `public/common/components/formats/nexus-repository-raw/.../internal/RawFormat.java`
- Recetas: `.../raw/internal/recipe/RawHostedRecipe.java`, `RawProxyRecipe.java`, `RawGroupRecipe.java`
- Handler HTTP: `.../raw/internal/recipe/RawContentHandler.java`
- Seguridad: `.../raw/internal/RawFormatSecurityContributor.java`
- Ejemplo complejo de metadatos: `.../maven/internal/group/RepositoryMetadataMerger.java`

## 5. Registro en el build

1. Añadir el módulo en `public/common/components/formats/pom.xml`:
   ```xml
   <module>nexus-repository-npm</module>
   ```
2. Añadir la dependencia en la assembly
   `public/selfhosted/assemblies/nexus-repository-core/pom.xml`:
   ```xml
   <dependency>
     <groupId>org.sonatype.nexus.common.components</groupId>
     <artifactId>nexus-repository-npm</artifactId>
   </dependency>
   ```
3. El descubrimiento de plugins es por classpath + anotaciones (`@Named`, `@Inject`), no hay
   registro manual adicional.

## 6. Fases y criterios de aceptación

### F1 — Hosted (MVP) — COMPLETADA (commit `6d22cfc`)
- [x] El módulo compila y aparece en la assembly.
- [x] `POST /service/rest/v1/repositories/npm/hosted` (o UI) crea el repo `npm-hosted`.
- [x] `npm publish --registry http://localhost:8081/repository/npm-hosted/` funciona.
- [x] `npm install <paquete>` y `npm view <paquete>` funcionan contra ese registry.
- [x] `npm dist-tag` add/rm/ls funciona.
- [x] Los assets se guardan con `sha1`/`sha512` y se pueden descargar por URL directa.

Notas de implementación:
- El packument se genera dinámicamente; el manifiesto por versión se guarda en `/{pkg}/{version}`.
- `latest` se deriva de la versión estable más alta; un prerelease publicado con `--tag` no lo mueve.
- Se soportan paquetes con scope y `_rev` para `npm unpublish` (REST `-rev` y CLI con `--force`).
- Repo hosted con `writePolicy: ALLOW` (con `ALLOW_ONCE` falla el segundo publish).

### F2 — Proxy — COMPLETADA (commit `3812074`)
- [x] `npm-proxy` cachea packuments y tarballs de `https://registry.npmjs.org`.
- [x] Se reescriben las URLs `dist.tarball` para apuntar al proxy.
- [x] `npm install` a través del proxy funciona en frío y en caliente.

Notas de implementación:
- Metadatos con `metadataMaxAge`; tarballs con `contentMaxAge`.
- Metadata upstream solicitada con `Accept-Encoding: identity`; la reescritura de `dist.tarball`
  ocurre en la respuesta, no en el asset almacenado.
- Paquetes scoped: metadata upstream con `%2F`; tarballs con layout normal.
- Negative cache para paquetes inexistentes; REST V1/Beta para crear/leer/actualizar el proxy.

### F3 — Group — COMPLETADA (commit `a793abb`)
- [x] `npm-group` agrega miembros hosted+proxy; resuelve versiones y dist-tags.

Notas de implementación:
- Merge de packuments y `dist-tags` en orden de miembros (gana el primero).
- `latest` se deriva del conjunto fusionado si ningún miembro lo define.
- Las URLs de tarball se reescriben al repositorio group.
- Manifiestos/tarballs/endpoints restantes usan el primer miembro que responde (hosted → proxy).
- Group de solo lectura (PUT/DELETE → 405); REST V1/Beta para crear/leer/actualizar el group.

### F4 — Extras — COMPLETADA
- [x] `/-/v1/search` en hosted, proxy y group.
- [x] `npm audit` (`/-/npm/v1/security/audits/quick` y `advisories/bulk`); el proxy reenvía al
  upstream.
- [x] Tarea `repository.npm-rebuild-metadata` para reconstruir metadatos de versiones.
- [x] UI de recetas y eliminación del facet Pro `NpmConfiguration` del formulario `npm_proxy`.

Notas de implementación:
- `NpmComponentAttributes` copia `description`, `keywords`, `author`, `license`, `homepage` y
  `repository` a atributos `npm.*` del componente y del asset del manifiesto; `/v1/search` los usa.
- El search recorre componentes (con `withGroupMemberContent()` en groups), agrupa por paquete y
  devuelve la versión más alta; limita el escaneo a 10 000 componentes y pagina con `from`/`size`.
- Datos legacy: antes el manifiesto `name/version` se guardaba como componente propio
  (`namespace=name`, `name=version`). `NpmContentFacetImpl.put` ya adjunta el manifiesto al
  componente del tarball; search y rebuild ignoran esos componentes legacy.
- El rebuild relee el manifiesto de cada versión, reaplica atributos y reporta
  `N versions updated, M versions without metadata`.
- `npm audit` hosted responde con un informe vacío si no hay advisories locales; el proxy reenvía
  el cuerpo al upstream y devuelve su JSON tal cual.

Verificación realizada:

- Publicación unscoped (`nexus-search-demo@1.0.0`) y scoped (`@demo/nexus-search-scoped@2.1.0`);
  un solo componente por paquete con manifiesto y tarball como assets.
- `/-/v1/search?text=nexus` devuelve ambos paquetes con descripción, keywords y publisher.
- `audits/quick` y `advisories/bulk` en hosted → 200; en proxy → advisories reales de npmjs
  (p. ej. `lodash@4.17.20`, CVE-2021-23337).
- Tarea de rebuild ejecutada por REST/script → `2 versions updated, 0 versions without metadata`.

### PostgreSQL y Pro — PostgreSQL IMPLEMENTADO

- El datastore público ya soporta PostgreSQL (driver `org.postgresql:postgresql` 42.7.2 como
  dependencia opcional, `MyBatisDataStore` detecta `jdbc:postgresql`, SQL por `databaseId`).
- El driver se empaquetó en la assembly y se añadió `DataStoreConfigurationEnvironmentSource`
  para configurar por `NEXUS_DATASTORE_NEXUS_*`. Verificado: 86 tablas, `databaseId: PostgreSQL`.
  Guía completa en `docs/POSTGRESQL.md`.
- Pro/Pro+ no se puede portar (código propietario); cualquier función equivalente debe
  reimplementarse desde especificaciones públicas. Lista priorizada en
  `docs/FUNCIONALIDADES_PRO.md`.

## 7. Riesgos y notas

- El packument de paquetes grandes (p. ej. `react`) es enorme; hay que generarlo de forma
  eficiente y con `ETag`/`If-None-Match`. **Pendiente de optimizar/medir.**
- `PUT /{package}` requiere autenticación de escritura y validar que el usuario tenga permiso
  sobre el repositorio. **Resuelto vía `SecurityFacet`/handler de seguridad de Nexus.**
- npm firma `dist.integrity` (sha512); no recalcular si viene, pero verificar coherencia.
  **El checksum se conserva y se sirve desde el blob.**
- No portar código del plugin npm de Community Edition: es propietario. Implementación limpia
  basada en la especificación pública del registro npm y en el patrón del plugin `raw`.
- El group es de solo lectura; la escritura distribuida (`supportsGroupWrite` en la UI) es una
  función Pro y no está implementada.
- El driver de PostgreSQL ya está empaquetado y verificado; ver `docs/POSTGRESQL.md`.
