# Estado del proyecto — Fork de Nexus Repository

> Documento vivo. Actualizar al final de cada sesión de trabajo.
> Última actualización: 2026-10-24

## 1. Objetivo

Crear un **fork de Nexus Repository Core** (`sonatype/nexus-public`, licencia EPL-1.0) que incorpore
las funcionalidades que hoy solo están en las ediciones completas de Nexus (Community Edition y,
cuando sea viable, Pro/Pro+), manteniendo el control del código y pudiendo modernizar el stack.

Alcance decidido por el usuario: **Opción 1 — forkear el núcleo y desarrollar los formatos/funciones
que falten como plugins**.

## 2. Decisiones tomadas

| Fecha | Decisión | Motivo |
|---|---|---|
| 2026-10-22 | Forkear `nexus-public` en lugar de reimplementar desde cero | Ahorra años de trabajo en el núcleo (API REST, UI, RBAC, tareas, blobs) |
| 2026-10-22 | Opción 1: desarrollar los formatos faltantes como plugins | El repo público solo trae Maven, Raw y APT; npm/Docker/PyPI/etc. no están en el código |
| 2026-10-22 | Descartado REA (reverse-engineer-anything) para esta fase | No hace falta para un fork; solo para documentar comportamiento/APIs no documentadas |

## 3. Hallazgo clave: qué contiene realmente el repo OSS

Evidencia del README oficial de `sonatype/nexus-public`:

> "This is the open source codebase of Nexus Repository Core. This contains functionality for
> **maven, raw, and APT** repository formats, and uses an **embedded H2 database** that is
> appropriate for small workloads."

Verificado en el árbol del repositorio (10.861 rutas, commit `0a8a425`):

- `public/common/components/formats/` contiene solo:
  - `nexus-repository-maven`
  - `nexus-repository-apt`
  - `nexus-repository-raw`
- Búsquedas en el código: `npm` → 0, `docker` → 0, `pypi` → 0, `helm` → 0, `yum` → 0,
  `nuget` → 0 (solo una pantalla de UI).

### Comparativa de ediciones

| Nivel | Contenido | ¿Forkeable? |
|---|---|---|
| `nexus-public` (Core) | Maven, Raw, APT + H2, UI, API REST, RBAC, tareas, LDAP | Sí (EPL-1.0) |
| Community Edition (binario gratis) | Añade npm, Docker, NuGet, PyPI y otros + PostgreSQL externo. Límites: 40.000 componentes y 100.000 req/día | No |
| Pro / Pro+ | HA/clustering, SAML/OIDC SSO, blobs S3/Azure/GCS, replicación DR, rollback de borrados, FIPS, staging, content replication; Pro+ añade cuarentena de malware, políticas de licencia, control de dependencias AI/MCP | No (propietario) |

**Conclusión:** para igualar la edición completa hay que implementar como plugins, al menos:
npm, Docker/OCI, PyPI, NuGet y Yum, además de soporte de PostgreSQL externo y features tipo Pro.
**Estado del fork (2026-10-23): npm (hosted/proxy/group y extras), Yum (hosted/proxy), PyPI
(hosted/proxy/group), Helm, Go, RubyGems y Docker/OCI (proxy pull-through) ya están implementados y
desplegados en la assembly; PostgreSQL externo está soportado y verificado. NuGet sigue pendiente.**

## 4. Entorno y workspace

- Máquina: Ubuntu 26.04.1 LTS, `x86_64`, `/home` con ~71 GB libres.
- Herramientas presentes: `git 2.53.0`, `node v24.21.0`, `corepack 0.36.0`.
- **No hay `sudo` sin contraseña**, así que el toolchain se instala localmente en `toolchain/`
  (sin tocar el sistema).
- Red: el DNS resuelve de forma intermitente y a veces solo devuelve IPv6 (sin ruta).
  Forzar IPv4 con `curl -4` funciona; Maven Central es accesible por IPv4.
  Existe un PAC corporativo en `/home/ladmin/Downloads/proxy.pac` (`HTTPS internet.etecsa.cu`)
  que **no resuelve**, por lo que de momento se trabaja sin proxy.

### Estructura del workspace

```
/home/ladmin/Downloads/nexus-fork/
├── ESTADO_PROYECTO.md          # este documento
├── toolchain/
│   ├── jdk-21.0.13+9/          # Temurin 21 (JAVA_HOME)
│   └── apache-maven-3.9.16/    # Maven (no hay mvnw en el repo)
└── nexus-public/               # clon del repo (--depth 1, --single-branch)
```

Instancia local de trabajo (2026-10-23): build en
`nexus-public/public/selfhosted/assemblies/nexus-repository-core/target/assembly/`, arrancada en
`http://localhost:8081` con los repositorios `npm-hosted`, `npm-proxy` y `npm-group` creados y
poblados con datos de prueba. Datos y logs en
`nexus-public/public/selfhosted/assemblies/nexus-repository-core/target/sonatype-work/nexus3/`.
El servidor se detiene con `kill <pid>` (no usar `bin/nexus stop`).

## 5. Progreso realizado

1. Clonado `https://github.com/sonatype/nexus-public.git` con `--depth 1 --single-branch`.
   - Commit base: `0a8a425` (2026-02-04) — "Merge pull request #906 from sonatype/sync-3.89-to-main".
   - Versión en `pom.xml`: `3.89.0-09`.
   - Tamaño del clon: ~91 MB. **Es shallow**: ejecutar `git fetch --unshallow` cuando la red lo permita.
2. Descarga de JDK 21 (Temurin `21.0.13+9`, Adoptium) y Apache Maven `3.9.16` en `toolchain/`.
   - Verificado: `java -version` y `mvn -v` funcionan con `JAVA_HOME=toolchain/jdk-21.0.13+9`.
3. Verificado que **el repo no incluye `mvnw`** (ni `.mvn/`) pese a lo que dice el README,
   por lo que se usa el Maven instalado en `toolchain/`.
4. Confirmado que Maven Central responde por IPv4 (`repo1.maven.org` → 200).
5. Preparado Yarn para el build del frontend:
   - El repo **no trae** `.yarn/releases/` aunque `.yarnrc.yml` apunta a `.yarn/releases/yarn-4.9.1.cjs`.
     Se descargó ese archivo desde `https://repo.yarnpkg.com/4.9.1/packages/yarnpkg-cli/bin/yarn.js`.
   - Shims de corepack instalados en `toolchain/bin` (`yarn`, `pnpm`, etc.) para no usar `sudo`.
   - `yarn install`: **completado** en 4m 24s, `node_modules` ≈ 671 MB, solo warnings de peer deps.
6. **Build completado con éxito** (2026-10-22 21:33):
   ```
   mvn -B clean install -Ppublic -DskipTests -Dmaven.javadoc.skip=true
   ```
   - 121 módulos, `BUILD SUCCESS`.
   - Único aviso relevante: `overlay does not exist` en la assembly (no impide el build).
   - Artefacto: `public/selfhosted/assemblies/nexus-repository-core/target/assembly/` (~155 MB).
7. **Arranque verificado**:
   - `bin/nexus` no venía con permiso de ejecución; se aplicó `chmod +x bin/nexus bin/setenv`.
   - Arrancó como `Sonatype Nexus CORE 3.89.0-09 (0a8a425d)` en **12,6 s**.
   - `GET http://localhost:8081/service/rest/v1/status` → **200**.
   - Credenciales iniciales en `.../target/sonatype-work/nexus3/admin.password`.
   - `bin/nexus stop` no funciona (falla al cargar clases); detener con `kill <pid>`.
   - La instancia de trabajo queda arrancada en el puerto 8081 para las pruebas F1-F3.
8. **Plugin npm F1 (hosted) completo y verificado** (2026-10-23). Ver detalle en 5.1.
9. **Plugin npm F2 (proxy) implementado y verificado** (2026-10-23). Ver detalle en 5.2.
10. **Plugin npm F3 (group) implementado y verificado** (2026-10-23). Ver detalle en 5.3.
11. **Evaluación de PostgreSQL/Pro** (2026-10-23). Ver detalle en 5.4.

### 5.1. F1 — npm hosted (completo)

Commit `6d22cfc` — `npm: add format module with publish, packument, dist-tags and unpublish support`.

**Módulo y registro en el build**

- Nuevo módulo `public/common/components/formats/nexus-repository-npm/`.
- Registrado en `public/common/components/formats/pom.xml`.
- Versión añadida al `dependencyManagement` de `public/common/components/pom.xml`.
- Dependencia añadida a `public/selfhosted/assemblies/nexus-repository-core/pom.xml`.

**Clases principales**

| Área | Clases |
|---|---|
| Formato/seguridad | `NpmFormat`, `NpmFormatSecurityContributor`, `NpmSecurityFacet` |
| Facet de contenido | `NpmContentFacet`, `NpmContentFacetImpl` |
| Receta/handler | `NpmRecipeSupport`, `NpmHostedRecipe`, `NpmContentHandler`, `NpmPackumentBuilder` |
| Dist-tags | `NpmDistTagsManager` |
| Stores MyBatis | `NpmAssetDAO`, `NpmComponentDAO`, `NpmContentRepositoryDAO`, `NpmAssetBlobDAO`, `NpmStoresProvider`, `NpmBrowseNodeDAO` |
| Coordenadas/semver | `NpmCoordinatesHelper` |
| REST hosted | `NpmHostedRepositoriesApiResource` (V1 + Beta), `NpmHostedRepositoryApiRequest`, converter, `NpmHostedApiRepository`, `NpmAttributes`, `NpmRepositoryAdapter` |
| Configuración | `NpmConfiguration` (bootstrap) |

**Comportamiento implementado**

- `npm publish` real: el handler extrae `_attachments`, guarda el tarball en
  `{paquete}/-/{fichero}.tgz` y registra el manifiesto de la versión en `{paquete}/{version}`.
- Packument **dinámico**: se genera desde componentes y manifiestos por versión; los assets se
  publican con slash inicial (`/demo-npm`, `/demo-npm/1.0.2`, `/demo-npm/-/demo-npm-1.0.2.tgz`).
  Si no hay datos dinámicos, se sirve el packument legacy almacenado en `/{paquete}`.
- `GET /{paquete}`, `GET /{paquete}/{version}`, `GET/PUT/DELETE` de dist-tags y descarga de tarball.
- `GET /-/whoami` y `GET /-/ping` para las CLIs de npm.
- Paquetes con scope (`@scope/name`, URL encoded `%40scope%2Fname`) en publish, packument,
  manifiesto y tarball.
- Checksums `sha1` y `sha512` servidos desde el blob (`AssetBlob.checksums()`).
- Dist-tags persistidos como atributo de componente `dist-tags`; `PUT` y `DELETE` por tag.
- `latest` se deriva de la **versión estable más alta** cuando no está fijado explícitamente;
  un `npm publish --tag beta` de un prerelease no mueve `latest`.
- `_rev` del packument (`1-<hex de nº de versiones>`) para que `npm unpublish` funcione.
- Unpublish soportado por `DELETE /{paquete}/-rev/{rev}` y por `npm unpublish <pkg> --force`;
  elimina componentes y assets del repositorio.
- Lectura anónima habilitada (la instancia ya tiene anónimo activo).
- El repositorio hosted debe crearse con `writePolicy: "ALLOW"`; con `ALLOW_ONCE` el segundo
  publish falla con 400.

**Verificación end-to-end realizada**

- Crear repo `npm-hosted` por REST → 201.
- Publicar `demo-npm@1.0.2`, `1.1.0-beta.1` (tag beta), paquete con scope
  `@demo-scope/scoped-pkg@1.0.0`.
- `npm install demo-npm@1.0.2` y `npm install demo-npm@beta` con caché limpia → OK y
  ejecución del módulo correcta.
- `npm install @demo-scope/scoped-pkg` → OK.
- Packument con versiones, `dist-tags` y `dist.integrity`/`shasum`/`tarball` correctos.
- `PUT`/`DELETE` de dist-tag (`beta`) → 201/204.
- Unpublish por REST `-rev` → 204 y 404 posterior; `npm unpublish --force` → OK.
- Tests unitarios de `NpmCoordinatesHelper` (parseo, orden semver, `isPrerelease`,
  `highestStableVersion`) en verde.

### 5.2. F2 — npm proxy (completo)

Commit `3812074` — `npm: add proxy repository with metadata and tarball caching`.

**Clases nuevas**

- `NpmProxyRecipe`: receta `npm-proxy` con handlers de proxy estándar.
- `NpmProxyFacet`: caché y URL upstream.
- `NpmProxyHandler`: reescritura de `dist.tarball` en la respuesta.
- REST: `NpmProxyRepositoriesApiResource` (V1 + Beta), `NpmProxyRepositoryApiRequest`,
  converter, `NpmProxyApiRepository`; rama `ProxyType` en `NpmRepositoryAdapter`.

**Comportamiento implementado**

- `npm-proxy` cachea metadatos y tarballs de `https://registry.npmjs.org`.
- Metadatos usan el **metadata cache controller**; tarballs el **content cache controller**.
- El metadata upstream se pide con `Accept-Encoding: identity` para poder reescribir el JSON.
- `dist.tarball` se reescribe **solo en la respuesta** para apuntar al proxy; el documento
  almacenado conserva las URLs originales upstream.
- Paquetes con scope: el metadata upstream se pide como `@scope%2Fname`; el path del tarball
  conserva su layout normal.
- Negative cache activa para paquetes inexistentes (404 estable).
- API REST para crear/leer/actualizar repos proxy npm (`/service/rest/v1/repositories/npm/proxy`).

**Verificación end-to-end realizada**

- Crear `npm-proxy` con `remoteUrl=https://registry.npmjs.org`, `contentMaxAge=1`,
  `metadataMaxAge=1` → 201.
- Instalación en frío de `is-number` → OK; packument con 15 versiones y
  `dist.tarball` apuntando a `http://localhost:8081/repository/npm-proxy/...`.
- Segunda instalación (caliente) → OK, servida desde caché.
- `is-odd` (con dependencia transitiva de `is-number`) → OK.
- Scoped `@isaacs/string-locale-compare` en frío y caliente → OK, metadata `%2F` incluida.
- Paquete inexistente → 404 en ambas llamadas (negative cache).
- Assets verificados en el proxy: packuments, manifiestos por versión y tarballs.

### 5.3. F3 — npm group (completo)

Commit `a793abb` — `npm: add group repository with packument and dist-tag merging`.

**Clases nuevas**

- `NpmGroupRecipe`: receta `npm-group`.
- `NpmGroupHandler`: merge de packuments y dist-tags.
- REST: `NpmGroupRepositoriesApiResource` (V1 + Beta), `NpmGroupRepositoryApiRequest`,
  converter, `NpmGroupApiRepository`; rama `GroupType` en `NpmRepositoryAdapter`.

**Comportamiento implementado**

- `npm-group` agrega miembros hosted+proxy (orden de `memberNames` = prioridad).
- Packuments fusionados: unión de `versions`; ante conflicto gana el primer miembro.
- `dist-tags` fusionados del mismo modo; si ningún miembro define `latest`, se deriva de la
  versión estable más alta del conjunto fusionado.
- Las URLs `dist.tarball` fusionadas se reescriben al repositorio group, de modo que el cliente
  descarga siempre a través del group y no depende de un miembro concreto.
- `GET /-/package/{pkg}/dist-tags` también se fusiona entre miembros.
- Manifiestos, tarballs y demás endpoints se sirven del primer miembro que responde
  correctamente (fallback hosted → proxy).
- Escritura en el group no soportada (DELETE/PUT → 405); el group es de solo lectura.
- API REST para crear/leer/actualizar repos group npm (`/service/rest/v1/repositories/npm/group`).

**Verificación end-to-end realizada**

- Crear `npm-group` con miembros `["npm-hosted", "npm-proxy"]` → 201.
- `demo-npm` (solo hosted): packument correcto y `latest=1.0.2`; se fusionó además una versión
  `0.0.1-security` existente en npmjs con el mismo nombre, sin alterar `latest` del hosted.
- `is-number` (solo proxy): 15 versiones, `latest=7.0.0`, tarball apuntando al group.
- Merge real: se publicó temporalmente `is-number@99.0.0` en hosted y el group mostró 16
  versiones, `latest=99.0.0` (prioridad hosted); el tarball `7.0.0` siguió resolviéndose por el
  proxy a través del group.
- `npm install is-number` → `99.0.0` (hosted); `npm install is-number@7.0.0` → `7.0.0` (proxy).
- Scoped `@isaacs/string-locale-compare` a través del group → OK.
- `GET /-/package/is-number/dist-tags` fusionado → `{"latest":"99.0.0"}`.
- Limpieza posterior: unpublish de `99.0.0`; el group volvió a `latest=7.0.0`.

### 5.4. Evaluación PostgreSQL / Pro

**PostgreSQL: soporte ya presente en el código público**

- `nexus-datastore-mybatis` ya declara `org.postgresql:postgresql` (42.7.2, `optional=true`).
- `MyBatisDataStore` detecta `jdbc:postgresql`, configura `org.postgresql.Driver` y usa HikariCP.
- Los DAO XML usan SQL específico por motor con `databaseId="PostgreSQL"` / `databaseId="H2"`
  (por ejemplo `KeyValueDAO.xml` y los stores de contenido).
- El perfil Maven `postgres-unit-tests` y `nexus-test-db` existen para pruebas contra PostgreSQL.
- `nexus-blobstore-s3` también está en el árbol público, así que el almacenamiento S3 no requiere
  código propietario.

**Bloqueos concretos para usar PostgreSQL**

1. **Driver no empaquetado:** el fat jar actual no incluye `postgresql-*.jar` en `BOOT-INF/lib`.
   Hay que añadir la dependencia `org.postgresql:postgresql` a la assembly (o quitar `optional`
   del módulo mybatis). Es un cambio de POM, no de código.
2. **Instancia dedicada:** no hay PostgreSQL propio. Docker está disponible y existe la imagen
   `postgres:15-alpine`; el puerto 5433 está ocupado por `rag-postgres` (ajeno al proyecto), así
   que hay que usar otro puerto (p. ej. 5434) y una base/credenciales nuevas.
3. **Configuración:** definir `nexus.datastore.nexus.jdbcUrl`, usuario y contraseña por propiedades
   de sistema o variables de entorno (`NEXUS_DATASTORE_NEXUS_JDBCURL`, etc.) antes de arrancar.
4. **Datos:** la base H2 actual (`sonatype-work/nexus3/db`) no se migra automáticamente a
   PostgreSQL. Para una instancia nueva hay que recrear repositorios y volver a publicar; para
   conservar datos haría falta una migración H2→PG (no incluida en este fork).
5. **Validación:** ejecutar el build y la suite con el perfil `postgres-unit-tests` y repetir las
   pruebas F1-F3 contra la instancia con PostgreSQL.

**Pro / Pro+: alcance y límites**

- No se puede portar código propietario de Pro/Pro+. Cualquier función equivalente debe
  reimplementarse desde especificaciones públicas y con licencia EPL-1.0.
- Ya existen en el árbol público piezas habitualmente asociadas a ediciones completas:
  `nexus-blobstore-s3`, tareas de limpieza, browse/search de contenido y las pantallas CoreUI
  `NpmHosted.js`, `NpmProxy.js`, `NpmGroup.js` (las recetas npm ya tienen UI).
- Funciones Pro que requerirían diseño propio: HA/clustering, SSO SAML/OIDC, replicación DR,
  rollback de borrados, FIPS, staging, cuarentena de malware, políticas de licencia y control de
  dependencias. Es un esfuerzo mucho mayor que los formatos y no debe mezclarse con F1-F4.
- Recomendación: cerrar primero F4 (extras npm) y PostgreSQL; después priorizar una a una las
  funciones Pro según necesidad real, documentando cada una como implementación limpia.

### 5.5. Commits del plugin npm

| Commit | Contenido |
|---|---|
| `6d22cfc` | F1: módulo npm, publish, packument dinámico, manifiestos, dist-tags, unpublish, tests |
| `3812074` | F2: proxy npm, caché metadata/contenido, reescritura de tarballs, REST proxy |
| `a793abb` | F3: group npm, merge de packuments/dist-tags, reescritura al group, REST group |

Estado del árbol: limpio tras el commit de F3 (`git status --short` sin cambios).

## 6. Bloqueos actuales

- **Ninguno crítico para F1-F3.** El toolchain compila, el núcleo arranca y npm hosted, proxy y
  group funcionan de extremo a extremo.
- **PostgreSQL:** soportado y verificado (driver empaquetado y configuración por entorno); ver
  5.7 y `docs/POSTGRESQL.md`. La migración H2→PostgreSQL no está automatizada.
- **Docker Hub/GCR:** el proxy funciona, pero los blobs de esos dos mirrors están geobloqueados en
  esta red (CloudFront/Google Storage → 403). GHCR y Quay funcionan completos; ver `docs/DOCKER.md`.
- **Pro/Pro+:** no se puede reutilizar código propietario; solo reimplementación limpia (5.4).
- **Red inestable:** DNS intermitente y respuestas solo-IPv6 para algunos hosts.
  Usar `curl -4`, `NODE_OPTIONS=--dns-result-order=ipv4first` y reintentos.
- **Sin `mvnw` en el repo:** usar siempre `toolchain/apache-maven-3.9.16/bin/mvn`.
- El `package.json` declara el workspace `private/cloud/components/nexus-cloudui-plugin`,
  que no existe en el clon público; Yarn lo toleró, pero conviene limpiarlo en el fork.
- El clon sigue siendo **shallow**; `git fetch --unshallow` falla por cortes de GitHub. No
  bloquea el desarrollo.

## 7. Cómo construir y ejecutar (repetible)

```bash
cd /home/ladmin/Downloads/nexus-fork/nexus-public
export JAVA_HOME=/home/ladmin/Downloads/nexus-fork/toolchain/jdk-21.0.13+9
export PATH=/home/ladmin/Downloads/nexus-fork/toolchain/bin:$JAVA_HOME/bin:/home/ladmin/Downloads/nexus-fork/toolchain/apache-maven-3.9.16/bin:$PATH
export MAVEN_OPTS="-Xmx3g -Djava.net.preferIPv4Stack=true"

yarn install

mvn -B clean install -Ppublic -DskipTests -Dmaven.javadoc.skip=true

cd public/selfhosted/assemblies/nexus-repository-core/target/assembly
chmod +x bin/nexus bin/setenv
./bin/nexus run     # http://localhost:8081
```

### 7.1. Repositorios npm de prueba

Con la instancia arrancada y anónimo activo, los tres repositorios se crean por REST:

```bash
AUTH='admin:<password>'
BASE=http://localhost:8081/service/rest/v1/repositories/npm

# hosted
curl -u "$AUTH" -H 'Content-Type: application/json' -X POST "$BASE/hosted" -d '{
  "name": "npm-hosted", "online": true,
  "storage": {"blobStoreName": "default", "strictContentTypeValidation": true, "writePolicy": "ALLOW"},
  "cleanup": {"policyNames": []}, "component": {"proprietaryComponents": false}, "npm": {}
}'

# proxy
curl -u "$AUTH" -H 'Content-Type: application/json' -X POST "$BASE/proxy" -d '{
  "name": "npm-proxy", "online": true,
  "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
  "proxy": {"remoteUrl": "https://registry.npmjs.org", "contentMaxAge": 1440, "metadataMaxAge": 1440},
  "negativeCache": {"enabled": true, "timeToLive": 1440},
  "httpClient": {"blocked": false, "autoBlock": true,
    "connection": {"retries": 0, "timeout": 60, "enableCircularRedirects": false,
                   "enableCookies": false, "useTrustStore": false}},
  "npm": {}
}'

# group (orden = prioridad)
curl -u "$AUTH" -H 'Content-Type: application/json' -X POST "$BASE/group" -d '{
  "name": "npm-group", "online": true,
  "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
  "group": {"memberNames": ["npm-hosted", "npm-proxy"]}, "npm": {}
}'
```

Uso con npm:

```bash
npm publish --registry http://localhost:8081/repository/npm-hosted/
npm install <paquete> --registry http://localhost:8081/repository/npm-proxy/
npm install <paquete> --registry http://localhost:8081/repository/npm-group/
```

Comprobaciones rápidas:

```bash
curl -s http://localhost:8081/repository/npm-group/<paquete> | python3 -m json.tool | head
curl -s http://localhost:8081/repository/npm-group/-/package/<paquete>/dist-tags
curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8081/repository/npm-group/<paquete>/-/<paquete>-<version>.tgz
```

Build rápido tras tocar solo el plugin:

```bash
mvn -o -q -pl public/common/components/formats/nexus-repository-npm test
mvn -o -B install -Ppublic -DskipTests -Dmaven.javadoc.skip=true   # regenera la assembly
```

## 8. Fork y acceso a GitHub

- Fork creado en la cuenta `luisrignack`: `git@github.com:luisrignack/nexus-public.git`.
- Remotos del clon:
  - `origin` → fork propio (SSH)
  - `upstream` → `https://github.com/sonatype/nexus-public.git`
- Clave SSH ed25519 generada en `.ssh/` (proyecto), sin passphrase:
  - Privada: `.ssh/id_ed25519_github` (600, ignorada por `.gitignore`)
  - Pública (para copiar): `CLAVE_PUBLICA_GITHUB.txt`
  - Config: `.ssh/config`; el repo usa `core.sshCommand` apuntando a esa config.
- Conexión verificada: `Hi luisrignack! You've successfully authenticated`.
- **Nunca subir `.ssh/` al repositorio.**
- `git fetch --unshallow --tags` **falló**: GitHub corta la conexión en transferencias grandes
  (`Recv failure`, `Failed to connect ...:443`). El clone sigue siendo shallow y ahora incluye
  los commits locales del plugin npm (F1-F3); el historial remoto profundo sigue pendiente.
  Reintentar más tarde con `--deepen=500` por tandas; no bloquea el desarrollo ni el build.

## 9. Documentos del proyecto

- `ESTADO_PROYECTO.md` — este documento (estado y bitácora).
- `docs/DISENO_PLUGIN_NPM.md` — diseño del plugin npm: endpoints, mapeo a componentes/assets,
  clases y fases F1-F4. F1-F4 completadas.
- `docs/DISENO_PLUGIN_YUM.md` — diseño del plugin Yum.
- `docs/DOCKER.md` — proxy Docker/OCI: prefijos, configuración, caché, seguridad y limitaciones.
- `docs/POSTGRESQL.md` — datastore PostgreSQL: variables, arranque, verificación y migración.
- `docs/FUNCIONALIDADES_PRO.md` — lista priorizada de funciones tipo Pro/Pro+ a reimplementar.

## 10. Próximos pasos

1. ~~F1 hosted mínimo~~ **hecho y verificado** (5.1).
2. ~~Completar F1: versiones, dist-tags, borrado, anónimo y tests~~ **hecho** (5.1).
3. ~~F2 proxy npm con caché de metadata/contenido y reescritura de tarballs~~ **hecho** (5.2).
4. ~~F3 group npm con merge de packuments/dist-tags~~ **hecho** (5.3).
5. ~~**F4 extras npm:** `/-/v1/search`, `npm audit`, tarea de rebuild y pulido de UI~~ **hecho**
   (5.6).
6. ~~**PostgreSQL:** driver en la assembly, instancia dedicada y verificación~~ **hecho** (5.7).
7. ~~**Docker/OCI:** proxy pull-through con mirrors de Daocloud~~ **hecho** (5.8).
8. **Pro:** ejecutar la lista priorizada de `docs/FUNCIONALIDADES_PRO.md`. ~~OIDC~~ **hecho**
   (entrada 2026-10-24); siguen UI del proxy Docker (hecha), GC de caché, SBOM y backup/restore.
9. **NuGet:** único formato principal que falta para cubrir los de la edición completa.
10. **UI:** verificar el formulario de repositorios con el bundle CoreUI regenerado y añadir la
    pantalla de administración del proxy Docker.
11. Retomar `git fetch --deepen=500` cuando la red lo permita y subir los commits al fork
    (`origin`).

## 11. Notas para el LLM

- Antes de proponer código, comprobar el estado real de la red y del toolchain (sección 4-6).
- No asumir que existen `java`, `mvn` ni `mvnw`.
- No asumir que el repo contiene plugins de formatos: verificar en
  `public/common/components/formats/` (hoy: maven, apt, raw y npm).
- El plugin npm vive en `public/common/components/formats/nexus-repository-npm/` y ya cubre
  hosted, proxy y group. Antes de añadir código, revisar las clases existentes (`NpmContentHandler`,
  `NpmProxyFacet`, `NpmGroupHandler`, `NpmPackumentBuilder`, `NpmDistTagsManager`,
  `NpmCoordinatesHelper`) para no duplicar lógica.
- Respetar la licencia EPL-1.0 y no portar código propietario de Pro/Pro+.
- Actualizar este documento al cerrar cada avance relevante.

## 2026-10-23 - F4 RPM/YUM (CentOS/Rocky/AlmaLinux/RHEL)
- Creado módulo `public/common/components/formats/nexus-repository-yum` (EPL-1.0). Registrado en `formats/pom.xml`, `components/pom.xml` y `nexus-repository-core` assembly.
- Parser RPM propio sin Redline (lee lead+signature+header, extrae NEVRA, summary/description, license/vendor/group/url/packager/buildhost/sourcerpm, files via DIRINDEXES/BASENAMES/DIRNAMES o OLDFILENAMES, provides/requires/conflicts/obsoletes con EVR/flags, changelog, tamaños y `header-range`). Usa `CountingInputStream` de Guava y validación de tamaños de header.
- Metadatos Yum: `YumMetadataBuilder` genera `repomd.xml`, `primary.xml.gz`, `filelists.xml.gz`, `other.xml.gz` (gzip determinista, checksums SHA-256 comprimido/sin comprimir, revisión por huella de paquetes). `YumMetadataFacet` mantiene caché y reconstruye al subir/borrar; almacena metadatos generados como assets con componente `_yum_metadata`.
- Hosted (`yum-hosted`): subida PUT de `.rpm` vía HTTP, eliminación DELETE, generación automática de repodata. Rechaza subida/eliminación de `repodata/`. Componentes RPM con namespace=arch, name=nombre, version=epoch:version-release, kind=`rpm`.
- Proxy (`yum-proxy`): passthrough con caché (funciona con mirrors RPM/repodata con rutas relativas; DNF resuelve contra baseurl).
- REST v1/beta para crear/actualizar repositorios `yum/hosted` y `yum/proxy`. Formato registrado en stores/browse.
- Validado manual: repo hosted + upload de RPM real, `repomd.xml` y `primary.xml.gz` generados correctamente; proxy a `dl.rockylinux.org` devuelve repomd upstream.
- Limitación: group yum no implementado (prioridad tras hosted/proxy). Sin `dnf`/`rpm` en host para validación completa con cliente, pero metadata generada compatible con formato Yum estándar.

## 2026-10-23 - F5 PyPI (hosted, proxy, group)
- Creado módulo `public/common/components/formats/nexus-repository-pypi` (EPL-1.0). Registrado en `formats/pom.xml`, `components/pom.xml` y `nexus-repository-core` assembly.
- Formato `pypi`, namespace vacío, componente por proyecto normalizado PEP 503. Atributos `pypi.requires_python`, `pypi.version`, `pypi.filename`.
- Hosted (`pypi-hosted`): upload multipart estilo twine en raíz, índices HTML dinámicos PEP 503 en `/simple/` y `/simple/{project}/`, descarga/borrado vía `/packages/{filename}`. Borrado por proyecto en índice.
- Proxy (`pypi-proxy`): cachea índices, reescribe enlaces de paquete a `/packages/{base64url(upstream-url)}/{filename}`, descarga vía facade con resolución directa a URL upstream. Redirige raíz a `simple/`.
- Group (`pypi-group`): fusiona índices de miembros (anclas extraídas, sin duplicados) respetando orden de miembros; archivos servidos por primer miembro exitoso.
- REST API v1/beta para crear/actualizar repositorios `pypi/hosted`, `pypi/proxy`, `pypi/group`.
- Compilado con JDK/Maven toolchain, desplegado en assembly y validado manual: upload a hosted, índices generados, proxy a `https://pypi.org/` con reescritura funcional, group fusiona contenido de hosted+proxy.

## 2026-10-23 - F4 npm: búsqueda, audit, rebuild y UI

- `/-/v1/search` implementado para hosted, proxy y group (`NpmSearchHandler`). Agrupa por paquete,
  elige la versión más alta, pagina con `from`/`size` (máx. 250) y limita el escaneo a 10 000
  componentes. En groups usa `withGroupMemberContent()`.
- Atributos `npm.*` (`description`, `keywords`, `author`, `license`, `homepage`, `repository`)
  escritos en componente y asset del manifiesto por `NpmComponentAttributes`.
- `npm audit`: `POST /-/npm/v1/security/audits/quick` y `/-/npm/v1/security/advisories/bulk`.
  En hosted responde informe vacío si no hay advisories locales; el proxy reenvía el body al
  upstream y devuelve su JSON.
- Tarea `repository.npm-rebuild-metadata` (`NpmRebuildMetadataTask`): relee manifiestos, reaplica
  atributos y reporta `N versions updated, M versions without metadata`.
- Corregido `NpmContentFacetImpl.put`: el manifiesto de versión se adjunta al componente del
  tarball en lugar de crear un componente `name=<versión>`. Search y rebuild ignoran componentes
  legacy de ese tipo (`isLegacyManifestComponent`).
- UI: eliminado el facet Pro `NpmConfiguration` del formulario `npm_proxy` en
  `RepositoryFormConfig.js`.
- Verificado: publish unscoped y scoped, search con descripción/keywords, audit hosted y proxy
  (advisories reales de `lodash@4.17.20`), rebuild por script REST y descarga de tarballs.

## 2026-10-23 - PostgreSQL externo

- Driver `org.postgresql:postgresql` añadido como `runtime` a la assembly: queda en
  `BOOT-INF/lib` del uberjar.
- Nuevo `DataStoreConfigurationEnvironmentSource` (`nexus-datastore-h2`) que lee
  `NEXUS_DATASTORE_NEXUS_JDBCURL/USERNAME/PASSWORD/SCHEMA/ADVANCED/TYPE` y sus equivalentes
  `nexus.datastore.nexus.*`, con prioridad sobre H2.
- Verificado con PostgreSQL 15 en el puerto 5434 y `KARAF_DATA` separado:
  `MyBatis databaseId: PostgreSQL`, 86 tablas creadas, Nexus en 8082 y status 200.
- Guía completa (contenedor, variables, arranque, verificación y migración) en
  `docs/POSTGRESQL.md`.
- Limitación: no hay migración automática H2→PostgreSQL; hay que recrear repositorios o exportar
  tabla por tabla.

## 2026-10-23 - Proxy Docker/OCI

- Nuevo módulo `nexus-repository-docker` (EPL-1.0), registrado en `formats/pom.xml`,
  `components/pom.xml` y la assembly.
- `DockerRegistryServlet` (`@WebServlet("/v2", "/v2/*")`) expone el API Registry v2 sin el prefijo
  `/repository/...`.
- `DockerRegistryService`: proxy pull-through con selección de upstream por primer segmento
  (`dockerhub`, `gcr`, `ghcr`, `quay`), fallback `library/` para imágenes oficiales de Docker Hub,
  tokens Bearer automáticos, streaming de blobs, `Range`, `ETag`/`If-None-Match` y caché en disco.
- `DockerCache`: blobs y manifests por digest inmutables; manifests por tag y tags con TTL
  (300 s por defecto); sidecars `.meta`; sirve copia stale si el upstream devuelve 5xx.
- Seguridad: `DockerConfiguration` registra cadenas Shiro para `/v2` y `/v2/**` (`nx-authc`,
  API key, `nx-anonymous`, anti-CSRF). Con el anónimo deshabilitado, `/v2/` responde 401 con
  challenge Basic; con el anónimo habilitado permite pull sin credenciales. Corregido el bypass
  que dejaba `/v2` fuera de la autenticación.
- Verificado: `docker login localhost:8081`, `docker pull localhost:8081/quay/prometheus/node-exporter:v1.7.0`
  y `docker pull localhost:8081/ghcr/stefanprodan/podinfo:6.0.0` completos; segunda descarga de un
  blob en ~7 ms desde caché.
- Limitación de red: los blobs de Docker Hub vía `docker.m.daocloud.io` redirigen a CloudFront y
  los de GCR a Google Storage; ambos responden 403 por geobloqueo. Los manifests sí funcionan.
  Detalle y alternativas en `docs/DOCKER.md`.
- Añadidos 27 tests unitarios (configuración, rutas, caché, challenge Bearer, Range y digests).

## 2026-10-23 - Otros formatos incorporados al árbol

Además de npm/Yum/PyPI, el árbol de trabajo contiene tres módulos completos (EPL-1.0) ya
integrados en la assembly:

- `nexus-repository-helm` — Helm (hosted, proxy, group).
- `nexus-repository-go` — módulos Go (hosted, proxy, group).
- `nexus-repository-rubygems` — RubyGems (hosted, proxy, group).

Los tres incluyen formato, facets de seguridad, stores REST v1/beta y convertidores, y se
empaquetan en `BOOT-INF/lib` del uberjar. Queda pendiente su verificación end-to-end con clientes
reales y documentarlos en `docs/` como se hizo con npm/Yum/PyPI.

## 2026-10-24 - Repositorios Docker en la UI

- Registrado el formato `docker` en el backend (`DockerFormat`) y la receta `docker-proxy`
  (`DockerProxyRecipe`), por lo que
  `GET /service/rest/internal/ui/repositories/recipes` ya devuelve
  `{"format":"docker","type":"proxy"}` y la opción aparece en **Create repository**.
- La receta adjunta `ConfigurableViewFacet`, `SecurityFacet`, `HttpClientFacet` y
  `DockerProxyFacet`; no crea conectores Jetty por repositorio. El formulario muestra la ruta
  `/v2/<repo>/<imagen>` en lugar de puertos.
- `DockerProxyFacetImpl` resuelve el upstream desde `dockerProxy.indexUrl` con fallback a
  `proxy.remoteUrl`, usa el mirror global `dockerhub` para `indexType=HUB` y descifra
  credenciales Basic/Bearer con `SecretsService`.
- `DockerRegistryService` resuelve el primer segmento tras `/v2/` como nombre de repositorio y
  cae al selector global de upstreams cuando no coincide.
- REST V1/Beta: `DockerProxyRepositoriesApiResourceV1` y `...Beta`, con
  `DockerProxyRepositoryApiRequest`, convertidor, `DockerRepositoryAdapter` y modelos
  `DockerAttributes`/`DockerProxyAttributes`.
- Añadidos los stores de contenido del formato (`DockerStoresProvider` y DAOs) porque
  `DefaultNormalizationPriorityService` exige un `FormatStoreManager` por cada `Format`.
- UI: `docker_proxy` usa `DockerIndexConfiguration`, `DockerForeignLayerConfiguration` y el nuevo
  `DockerRepositoryRoutingConfiguration`; se eliminaron los facets Pro de conectores y V1 API.
- Verificado end-to-end: receta visible, `POST /service/rest/v1/repositories/docker/proxy` → 201,
  `GET` del repositorio, manifest y blob por `/v2/docker-ghcr/astral-sh/uv/...` y
  `docker pull localhost:8081/docker-ghcr/astral-sh/uv:latest` completo.
- Tests del módulo: 29 (sin fallos). Documentación actualizada en `docs/DOCKER.md`.

## 2026-10-24 - Catálogo de funciones Pro actualizado

- `docs/FUNCIONALIDADES_PRO.md`: marcada como completada la UI/API de repositorios Docker
  (punto P0 #2) y añadida la sección 1.1 con el hallazgo de que las pantallas Pro de User Tokens,
  SAML, OAuth2, Crowd, Nodes, Licensing y Malware Remediation ya existen en `nexus-coreui-plugin`
  pero sin backend.
- Prioridad recomendada para el siguiente bloque: User Tokens (solo falta REST + DAO), luego GC de
  caché Docker, SBOM CycloneDX y backup/restore; OIDC como primer trabajo de identidad.

## 2026-10-24 - User Tokens (Pro), ruta UI LDAP y fix de indexación SQL

- **User Tokens (commit `eebed1c`):** implementado el backend completo de User Tokens que en las
  ediciones comerciales requiere licencia Pro:
  - Almacenamiento sobre `ApiKeyService`, `UserTokenService` / `UserTokenServiceImpl`,
    `UserTokenRealm`, `UserTokenAuthenticationTokenFactory`, `UserTokenEnforcementFilter` y
    `UserTokenCapability` (+ descriptor/booter/configuration).
  - Recursos REST (`UserTokenApiResourceV1`, `UserTokenApiResourceBeta`,
    `CurrentUserTokenApiResource`, `UserTokenResetApiResource`) y privilegios
    `usertoken-settings`, `usertoken-users` y `usertoken-current` registrados en
    `NexusSecurityContributor`.
  - Eliminado el gating `editions: ['PRO']` y `licenseValid` en `adminRoutes.js` y `userRoutes.js`.
  - Tests unitarios en `UserTokenAuthenticationTokenFactoryTest`, `UserTokenEnforcementFilterTest`
    y `UserTokenServiceImplTest`.
- **UI LDAP (commit `f76fd19`):** ocultada la ruta `/admin/security/ldap` cuando el bundle
  backend `nexus-ldap-plugin` no está activo para evitar errores de ExtJS/React al abrir la página.
- **SQL Search (commit `cfe3701`):** `SqlSearchUpdateService.needsReindex` y `doneReindexing`
  usan ahora `repository.optionalFacet(ContentFacet.class)` para omitir repositorios sin
  `ContentFacet` (como `docker-proxy`) sin fallar en el chequeo periódico de reindexación;
  test unitario `SqlSearchUpdateServiceTest` en verde (5/5).

## 2026-10-24 - SSO OIDC (Pro), P0 #1

- Implementado el SSO OAuth2/OIDC completo (realm + flujo + UI), la primera función de identidad
  de la lista `docs/FUNCIONALIDADES_PRO.md`:
  - **`nexus-security`:** `OidcRealm` (realm Shiro `OAuth2Realm`), `OidcAuthenticationToken` e
    interfaces `OidcConfiguration` / `OidcConfigurationStore`. Inyección con `@Lazy` en
    `SecuritySystem` y `OidcConfigurationStore` para romper el ciclo
    `OidcRealm → SecuritySystem → RealmManagerImpl → OidcRealm` (mismo patrón que
    `UserManagerImpl`); sin él, más de 100 tests de `nexus-security` fallaban al montar el
    contexto Spring.
  - **`nexus-core`:** `OidcService` (cliente OIDC con `java-jwt` 3.16.0: canje de code, validación
    de firma JWKS/`iss`/`aud`/`exp`/`nonce`, extracción de claims y grupos),
    `OidcConfigurationStoreImpl` (capability `oidc` + `SecretsService` con placeholder para no
    sobrescribir el secreto), `OidcLoginFlow` + `OidcLoginServlet`
    (`GET /oidc/login` → IdP, `GET /oidc/callback` → sesión, `state`/`nonce` de un solo uso con
    TTL de 5 min), `OidcConfigurationResource` (REST interna UI `GET/PUT
    /service/rest/internal/ui/oauth2`) y capability `OidcCapability` (+ descriptor/booter).
  - **`nexus-coreui-plugin`:** `OidcStateContributor` publica `oauth2Enabled`; ruta
    `/admin/security/oauth2` des-gated (solo `Permissions.OIDC_SETTINGS.READ`, sin
    `editions: ['PRO']` ni `bundle`); `Permissions.OIDC_SETTINGS` en `nexus-ui-plugin`. El botón
    SSO de la página de login navega a `/oidc/login?hash=<b64 returnTo>` y tras el callback se
    redirige al fragmento original.
  - **Privilegios:** `nx-oidc-settings-read` y `nx-oidc-settings-update` en
    `NexusSecurityContributor`.
  - **Provisioning JIT:** usuarios del IdP se crean en `UserManager.DEFAULT_SOURCE`; los grupos
    coincidentes con roles existentes se mapean bajo `OAuth2Realm` vía `setUsersRoles`.
- **Tests:** nuevos `OidcServiceTest` (8), `OidcConfigurationStoreImplTest` (7) y
  `OidcRealmTest` (6); `NexusSecurityContributorTest` actualizado (2). Suites completas en verde:
  `nexus-security` 362, `nexus-base` 151, `nexus-core` 368, frontend `nexus-ui-plugin` 61
  suites. Los 12 fallos jest preexistentes de `nexus-coreui-plugin` (usertoken/LDAP/licencias en
  `RepositoriesForm`, `SettingsSidebar`, `GlobalHeader`, `AdminSecurityDirectoryPage`) fueron
  verificados como ajenos mediante `git stash`.

## 2026-10-24 - Validación E2E de Helm, Go y RubyGems (paso 1)

- **Validación E2E completa en la instancia local** (`docs/VALIDACION_HELM_GO_RUBYGEMS.md`):
  los tres formatos funcionan end-to-end con artefactos reales (chart `.tgz` fabricado, módulo
  Go con `.mod/.info/.zip`, `.gem` manuales) y remotos reales (`proxy.golang.org`,
  `rubygems.org`, Bitnami).
- **Grupos sin `memberNames`** (`helm-group`, `go-group`, `gems-group`): configurados por
  `PUT /v1/repositories/<fmt>/group/<name>` (endpoint por formato; el genérico `/repositories/<name>`
  devuelve 405).
- **Defectos reales encontrados y corregidos** (commit `e4055d9`, build 11:42/11:50):
  1. `FluentAssetBuilderImpl` aplicaba `OVERLAY` a atributos escalares en re-publicaciones:
     go-hosted devolvía 500 al re-subir y los índices generados de helm/rubygems quedaban
     obsoletos para siempre (regeneración silenciosamente fallida). Fix: `SET` para valores no-Map,
     `OVERLAY` solo para mapas. 431 tests de `nexus-repository-content` en verde.
  2. `HelmIndexBuilder.parseIndex` exigía claves sin indentar (nunca matcheaba su propio formato
     ni Bitnami). Fix: parseo con **SnakeYAML 2.3** + límite de code points `Integer.MAX_VALUE`
     (el default de 3 MB rechazaba el índice de Bitnami de 27 MB). Dependencia agregada al pom.
- **Resultados finales tras reiniciar la instancia con la build corregida:**
  `helm-group2/index.yaml` = 4,5 MB con 19.117 versiones mergeadas (demo propio + Bitnami);
  `helm-group` recuperado; specs.4.8.gz rubygems regenerado con contenido; re-PUT de go 201;
  charts y gems byte-idénticos (md5). El índice Bitnami vía proxy pasa la 302 a
  `repo.broadcom.com` sin tocar nada.
- **Limitación externa documentada:** `rubygems.org` retiró `/api/v1/dependencies` (404
  passthrough correcto); `charts.helm.sh` ya no sirve índice.
- **Repos auxiliares de contraste** (`helm-hosted3`, `helm-group2`) quedan instalados como
  evidencia reproducible.
- **Push:** `origin/main` actualizado a `e4055d9`.

## 2026-10-24 - Saneamiento de seguridad, push al fork y commits documentados

- **Commits cerrados** (toda la suite tocada en verde):
  - `5552712` — GC de caché Docker: tarea `repository.docker-gc`.
  - `8913d17` — SBOM: exportación de inventario CycloneDX 1.5 / SPDX 2.3 vía REST (`nexus-sbom`).
  - `fe3e0e4` — Backup: tarea de backup de blobs y datastore con manifiesto verificado (`nexus-backup`).
  - `51add5a` — **SSO OIDC**: commiteado el trabajo que estaba sin guardar — 26 archivos, +3.145 líneas
    (`OidcRealm`, `OidcService` con java-jwt, `OidcLoginFlow`/`OidcLoginServlet`, capability,
    REST `internal/ui/oauth2`, ruta UI des-gated, provisioning JIT, privilegios
    `nx-oidc-settings-read/update`). Re-ejecutados en verde: `OidcServiceTest` (8),
    `OidcConfigurationStoreImplTest` (7), `OidcRealmTest` (6) y usertoken (30) en `nexus-core`.
- **Saneamiento de credenciales (commit `10aa594`):**
  - `admin-creds.txt` eliminado del tracking y añadido a `.gitignore`.
  - Historia reescrita con `git filter-branch`: purgada la contraseña del árbol y del mensaje del
    commit (ahora `[REDACTED]`), refs de respaldo eliminados, `reflog expire --expire=now` y
    `gc --prune=now`. Verificado: 0 apariciones de la contraseña en mensajes y árboles; el archivo
    no existe en ningún commit.
  - **La contraseña real la rotó el usuario** (decisión 2026-10-24).
- **Push:** `origin/main` (`luisrignack/nexus-public`) apunta a `10aa594` tras el history-rewrite
  (force push; antes `origin/main` estaba en `0a8a425`). Copia de seguridad pre-rewrite en
  `/tmp/opencode/nexus-pre-rewrite.bundle` (borrar cuando se confirme que ya no hace falta).
- El punto de partida upstream del fork sigue siendo `0a8a425` (3.89.0-09).

## 2026-10-24 - LDAP: realm, configuración y API REST de administración (commit `fa46a5e`)

- Implementado el backend completo de LDAP, la autenticación directorial que en las ediciones
  comerciales va en el plugin cerrado `nexus-ldap-plugin` (17 ficheros, +3.024 líneas):
  - **`nexus-security`:** `LdapRealm` (realm Shiro de autenticación), interfaces
    `LdapConfiguration` / `LdapConfigurationStore` / `LdapDirectory` y sus implementaciones.
  - **`nexus-coreui-plugin` (Java):** `LdapCapability` + descriptor + propiedades (clave `ldap`,
    credenciales cifradas con `SecretsService`), `LdapConfigurationData` y
    `LdapConfigurationStoreImpl` sobre `CapabilityRegistry`, `LdapDirectoryImpl` (bind, resolución
    de usuario, grupos, paginación y filtros; 699 líneas).
  - **REST:** `LdapServerApiResource` (`/service/rest/v1/security/ldap`), `LdapServerXO` y
    `LdapUserXO`.
  - **Privilegios:** `nx-ldap-{all,create,delete,read,update}` en `NexusSecurityContributor`
    (+ test actualizado).
  - Ajuste en `BackupFileManager` para que el backup incluya la nueva configuración.
- **Tests:** `RealmManagerAuthorizingRealmTest` y `NexusSecurityContributorTest` adaptados, en verde.

## 2026-10-24 - Pantalla LDAP conectada a la UI React (commit `2df79cf`)

- Cierre del fix de UI LDAP (6 ficheros, +694/−92): la ruta `/admin/security/ldap` apuntaba a un
  bundle `nexus-ldap-plugin` inexistente, así que la pantalla no aparecía ni funcionaba.
  - `adminRoutes.js`: retirado el gate `bundle` de `visibilityRequirements`; la ruta renderiza el
    componente React `LdapServers` con permisos reales.
  - `LdapServerApiResource`: añadido `GET /{name}` y resolución **por nombre** en
    `read`/`update`/`delete`/`change-order` (fallback al id del cuerpo para los renombres).
  - `LdapServerXoMapper.java` (nuevo): mapeo XO↔config compartido por el recurso REST y el
    componente Direct, con `order` null-guard.
  - `LdapServerComponent.java` (nuevo, `@DirectAction("ldap_LdapServer")`):
    `readTemplates`, `verifyConnection`, `verifyUserMapping`, `verifyLogin` (decodificación
    base64/ISO-8859-1 del `btoa` de la UI), `clearCache`; sustituye `PASSWORD_PLACEHOLDER` por el
    secreto almacenado y devuelve los errores con `Responses.error(...)` para que
    `ExtAPIUtils.checkForError` los traduzca a `success:false` + mensaje.
  - `LdapSchemaTemplateXO.java` (nuevo): plantillas **Active Directory**, **Posix Static**,
    **Posix Dynamic** y **Generic**.
  - `nexus-coreui-plugin/pom.xml`: dependencia explícita de `nexus-core`.
- **Verificación en vivo sobre la assembly:** `GET/PUT/DELETE /security/ldap/{name}` → 200/404/204
  (antes 405), `change-order` por nombre → 204, renombre → 200; `extdirect readTemplates` → 200 con
  4 plantillas (antes 500); `clearCache` → éxito; los tres `verify*` devuelven `success:false` con
  mensaje legible; el bundle servido contiene `admin.security.ldap` y **0** menciones de
  `nexus-ldap-plugin`; `api.js` registra `ldap_LdapServer`; privilegios `nx-ldap-*` presentes.
- **Tests:** `nexus-core` 368, `nexus-coreui-plugin` 164 (Java) y 43 tests React LDAP en verde;
  `yarn check-format` limpio. Documentación nueva: `docs/LDAP.md`.

## 2026-10-24 - Documentación local, redacción de credenciales y repo raíz

- **Redacción de credenciales en los docs** (previo a subir todo el proyecto): las 4 apariciones de
  `admin:<credencial>` en `docs/DOCKER.md` y la de `docs/VALIDACION_HELM_GO_RUBYGEMS.md` se
  sustituyeron por `admin:$NEXUS_PASSWORD`, con nota de que el valor vive en
  `nexus-public/admin-creds.txt` (ignorado por git). Barrido de secretos en el workspace fuera de
  `nexus-public` y `toolchain`: limpio.
- **`.gitignore` de la raíz ampliado:** `.ssh/`, `toolchain/`, tokens (`gh-token`, `.gh-token`,
  `*.token`), `*.bak`, `dep.bin`, `nexus-public/admin-creds.txt`, `node_modules/`, `target/`.
- **Documentación actualizada:**
  - `ANALISIS_PROYECTO.md`: estructura, estado de formatos (Helm/Go/RubyGems ya validados E2E),
    funciones incorporadas, lista de commits y próximos pasos.
  - `docs/FUNCIONALIDADES_PRO.md`: GC, SBOM y backup marcados como completados con su commit; fila
    de LDAP añadida; nueva **sección 8 (informe de lo que queda por incorporar)**.
  - `docs/LDAP.md` (nuevo): descripción completa de la función LDAP, verificación y límites.
  - `docs/DOCKER.md` y `docs/VALIDACION_HELM_GO_RUBYGEMS.md`: nota de credenciales.
- **Repo raíz del workspace:** `/home/ladmin/Downloads/nexus-fork` pasa a ser un repositorio git
  propio con la documentación del proyecto, registrando `nexus-public` como **submódulo** apuntando
  a `git@github.com:luisrignack/nexus-public.git`.

