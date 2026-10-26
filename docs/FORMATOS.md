# Formatos de repositorio del fork

Estado de los formatos soportados por **nexus-fork** (base: `sonatype/nexus-public` 3.89.0-09, EPL-1.0).
Todos los formatos nuevos se implementan desde las especificaciones públicas de cada protocolo:
no se ha portado código de las ediciones comerciales de Sonatype.

> Módulo Java: `nexus-public/public/common/components/formats/nexus-repository-<formato>`.
> Recetas: `<formato>-hosted`, `<formato>-proxy`, `<formato>-group` (las que apliquen).

## 1. Resumen

| Formato | Recetas | Estado | Módulo | Tests |
|---|---|---|---|---|
| `maven2`, `npm`, `pypi`, `raw`, `docker`, `apt`, `yum`, `go`, `helm`, `rubygems`, `r`, `conda`, `composer`, `huggingface`, `terraform` | hosted/proxy/group | ✅ upstream | propio | upstream |
| **`ansible`** | hosted + proxy + group | ✅ nuevo + E2E | `nexus-repository-ansible` | 31 |
| **`alpine`** | hosted + proxy + group | ✅ nuevo + E2E | `nexus-repository-alpine` | 9 |
| `nuget` | hosted + proxy + group | 🚧 en curso | `nexus-repository-nuget` | — |
| `cargo` | hosted + proxy + group | 🚧 en curso | `nexus-repository-cargo` | — |
| `conan` | hosted + proxy + group | 🚧 en curso | `nexus-repository-conan` | — |
| `cocoapods` | proxy | ⏳ pendiente | `nexus-repository-cocoapods` | — |
| `p2` | proxy | ⏳ pendiente | `nexus-repository-p2` | — |
| `swift` | hosted + proxy | ⏳ pendiente | `nexus-repository-swift` | — |
| `gitlfs` | hosted | ⏳ pendiente | `nexus-repository-gitlfs` | — |

También están registrados en `DatastoreHighAvailabilitySupportChecker` (todos los anteriores, incluidos
`ansible` y `alpine`) para que la alta disponibilidad los acepte.

## 2. Ansible (Galaxy collections)

Recetas: `ansible-hosted`, `ansible-proxy`, `ansible-group`.

### Endpoints

Bajo la base del repositorio (`http://host:8081/repository/<repo>/`):

| Método | Ruta | Respuesta |
|---|---|---|
| GET | `/api/` | descubrimiento Galaxy (`available_versions.v3`/`v1`) |
| GET | `/api/v3/` | raíz de la API (`collection_index`, `available_versions`) |
| GET | `/api/v3/collections/…` | alias v3 usados por `ansible-galaxy` 2.21+ |
| GET | `/api/v3/plugin/ansible/content/published/collections/index/` | todas las collections |
| GET | `…/index/{ns}/` | collections de un namespace |
| GET | `…/index/{ns}/{name}/` | detalle (`highest_version`, `dependencies`, `version_count`) |
| GET | `…/index/{ns}/{name}/versions/` | versiones (la más reciente primero) |
| GET | `…/index/{ns}/{name}/versions/{v}/` | detalle de versión (`artifact`, `manifest`, `download_url`) |
| GET | `/api/v3/collections/{ns}/{name}/{v}/download/` | tarball |
| GET | `/api/v3/plugin/ansible/content/published/collections/artifacts/{file}.tar.gz` | tarball de Galaxy público (proxy) |
| GET | `/api/v2/collections/{ns}/{name}/[versions[/ {v}]]` | compatibilidad Galaxy v2 |
| PUT | `…/versions/{v}/` (body = tar.gz) o `/collections/{ns}/{name}/{v}/{ns}.{name}-{v}.tar.gz` | publicación (hosted) |
| DELETE | `…/index/{ns}/{name}…` | borra la collection |
| GET | `/` | **200 `text/html`** (página de aterrizaje; no redirige) |

### Almacenamiento

```
collections/{namespace}/{name}/{version}/{namespace}.{name}-{version}.tar.gz
```

El `MANIFEST.json` del tarball es la fuente de verdad: si no coincide con las coordenadas de la URL se
responde **400**. Descripción, autores y dependencias se guardan como atributos del componente
(`ansible.description`, `ansible.authors`, `ansible.dependencies`).

### Por tipo

- **hosted**: lee/escribe el almacén y genera la API Galaxy desde los componentes almacenados.
- **proxy**: reenvía a `proxy.remoteUrl`, cachea (metadatos vs. contenido) y reescribe tanto las URLs
  absolutas como las rutas root-relative (`"/api/…"`) de las respuestas JSON para que apunten al
  repositorio local. Reenvía `limit`/`offset` y conserva el paginado de Galaxy; las peticiones con
  query no se responden desde la caché por ruta para no mezclar páginas. Los tarballs y los paths
  desconocidos conservan sus bytes.
- **group**: fusiona los documentos de los miembros (primera collection que aparezca gana), reescribe
  las URLs de cada miembro a las del group y sirve los tarballs desde el miembro que los tenga.

### Prueba rápida

```bash
BASE=http://localhost:8081/repository/ansible-hosted
curl -u admin:$NEXUS_PASSWORD -X POST http://localhost:8081/service/rest/v1/repositories/ansible/hosted \
  -H 'Content-Type: application/json' -d '{
    "name":"ansible-hosted","online":true,
    "storage":{"blobStoreName":"default","strictContentTypeValidation":false},"ansible":{}}'
curl -s $BASE/api/v3/
ansible-galaxy collection build            # genera community-general-<v>.tar.gz
curl -u admin:$NEXUS_PASSWORD -X PUT --data-binary @community-general-1.0.0.tar.gz \
  "$BASE/api/v3/plugin/ansible/content/published/collections/index/community/general/versions/1.0.0/"
```

### UI

- ExtJS: `view/repository/recipe/Ansible{Hosted,Proxy,Group}.js` + entradas en el array `views` de
  `controller/Repositories.js` + filtro de búsqueda `controller/SearchAnsible.js` registrado en
  `app/PluginConfig.js` (+ `PluginStrings.js`, `util/RepositoryUrls.js`).
- React: `Search/SearchAnsibleExt.jsx`, ruta `browse.search.ansible`, entrada en el menú lateral y
  cadenas en `SearchStrings.jsx`.

### Verificación E2E (2026-10-25, sobre la assembly real en `:8081`)

Ejecutado con `/tmp/opencode/e2e_ansible.sh` (fixtures en `/tmp/opencode/*.tar.gz`, upstream
Galaxy falso en `127.0.0.1:8099`): **todo en verde**.

| Comprobación | Resultado |
|---|---|
| Recetas en `GET /service/rest/internal/ui/repositories/recipes` | `['group','hosted','proxy']` |
| `POST /service/rest/v1/repositories/ansible/{hosted,proxy,group}` | 201 / 201 / 201 |
| `GET /` en hosted, proxy y group | **200 `text/html`** (página de aterrizaje) |
| `GET /api/` en hosted, proxy y group | 200 `available_versions` (`v3`, `v1`) |
| `PUT` por `…/index/{ns}/{n}/versions/{v}/` y por `/collections/{ns}/{n}/{v}/{n}.{n}-{v}.tar.gz` | 201 |
| `PUT` sin coordenadas / coordenadas ≠ `MANIFEST.json` / tarball que no es collection | 400 / 400 / 400 |
| Índice, namespace, detalle, versiones y detalle de versión (v3) | 200 (2 versiones, `manifest`, `artifact.sha256`) |
| `GET …/download` | 200 `application/x-gzip`, `gzip -t` OK; versión inexistente → 404 |
| Galaxy v2 (`/api/v2/collections/…`) | 200 |
| Group: índice fusionado con URLs reescritas a `ansible-group` + descarga | 200 + gzip OK |
| Proxy: índice con URLs reescritas a `ansible-proxy` + descarga del tarball | 200 + gzip OK |
| `DELETE …/index/{ns}/{n}` (y segundo DELETE) | 204 / 404 |

### Validación contra Galaxy real (2026-10-26)

La IP de este entorno está bloqueada por Cloudflare (`403 "Sorry, you have been blocked"`) para
`galaxy.ansible.com`; el bloqueo afecta también a `curl` directo y no a la lógica del proxy. Para
validar con datos reales se usó un relay local (`/tmp/opencode/galaxy_relay.py`, `127.0.0.1:8098`)
que obtiene las respuestas **en vivo** de `galaxy.ansible.com` a través de un proxy HTTP y solo
reescribe la autorreferencia `https://galaxy.ansible.com` → `http://127.0.0.1:8098`; los tarballs
pasan byte a byte. Nexus se configuró temporalmente con `remoteUrl=http://127.0.0.1:8098`.

| Comprobación | Resultado |
|---|---|
| `GET /api/` | 200 `{"available_versions":{"v3":"v3/","v1":"v1/"}}` |
| `GET /api/v3/collections/community/general/` | 200; `href` y `versions_url` reescritos a `http://localhost:8081/repository/ansible-galaxy-real/…` |
| `GET …/versions/?limit=100` | 200; `meta.count=243`, 100 versiones, `links.next` reescrito al repo local |
| Paginado `offset=100` y `offset=200` | 100 + 43 versiones, `next` local y `null` al final |
| `GET …/versions/13.5.0/` | 200; `download_url` local; `artifact.sha256=54598287…`, `size=2913455` |
| Descarga del artefacto por Nexus | 200, 2.913.455 bytes, `gzip -t` OK, sha256 `54598287e0ea0504aee2d32c884208ddf26f3ac41e0b4e773ff514af210787c4` |
| `ansible-galaxy collection install community.general:13.5.0` (ansible-core 2.21.5) | **OK**, más la dependencia `community.library_inventory_filtering_v1:1.1.5` |

El mismo flujo se probó antes con un upstream falso local que servía los documentos y el tarball
reales capturados de Galaxy; también terminó en verde. El relay solo fue un puente de red: el
contenido servido por Nexus es el de Galaxy real y el checksum del tarball coincide con el publicado.

### Limitaciones

- Los listados de repositorios **hosted/group** llevan la colección dos veces (`data` para v3 y
  `results` para v2) desde un único generador y devuelven `links.next=null` (listado completo).
  El **proxy** sí reenvía `limit`/`offset` y conserva el paginado real de Galaxy.
- `galaxy.ansible.com` devuelve **403 de Cloudflare** desde la IP de este entorno. La validación
  con Galaxy real se hizo con un relay local que usa un proxy HTTP solo como transporte; el
  contenido y el checksum son los de Galaxy. Para reproducirla sin relay hace falta una IP no
  bloqueada.

## 3. Alpine (apk)

Recetas: `alpine-hosted`, `alpine-proxy`, `alpine-group`. Módulo
`nexus-repository-alpine`. Soporta paquetes **APK v2** (gzip + tar + `.PKGINFO`); APK v3/ADB
todavía no.

### Layout y endpoints

```
<branch>/<repository>/<arch>/<paquete>-<version>.apk
<branch>/<repository>/<arch>/APKINDEX.tar.gz
```

Bajo la base del repositorio (`http://host:8081/repository/<repo>/`):

| Método | Ruta | Respuesta |
|---|---|---|
| GET/HEAD | `/<prefijo>/<arch>/APKINDEX.tar.gz` | índice generado (hosted/group) o cacheado (proxy) |
| GET/HEAD | `/<prefijo>/<arch>/<paquete>.apk` | paquete |
| PUT/POST | `/<prefijo>/<arch>/<paquete>.apk` | alta (solo hosted) |
| DELETE | `/<prefijo>/<arch>/<paquete>.apk` | borrado (solo hosted) |
| GET | `/` | página de aterrizaje `text/html` |

### Índice y checksum

El `C:` de cada estancia es el SHA1 de los **bytes comprimidos del miembro gzip de control**
(el que contiene `.PKGINFO`), en Base64 con prefijo `Q1`. Es exactamente lo que calcula
`apk index`; las estancias generadas coinciden byte a byte con las de `apk-tools` 2.14.x y 3.x
(verificado con `musl-1.2.5-r4.apk` y `aaudit-0.7.2-r3.apk`).

El hosted infla solo el miembro de control (nunca la sección de datos), guarda la estancia como
atributo del asset (`alpine.indexEntry`) y regenera `APKINDEX.tar.gz` en cada petición a partir de
los `.apk` del mismo directorio. El índice no va firmado.

### Por tipo

- **hosted**: valida el `.apk` al subirlo (`400` si no es APK v2), guarda el paquete y genera el
  índice del directorio. Un `PUT` sobre `APKINDEX.tar.gz` se rechaza con `400`.
- **proxy**: passthrough puro a `proxy.remoteUrl` (sin reescritura: el índice no tiene URLs
  autorreferentes), con `Accept-Encoding: identity` y caché separada de metadatos/contenido.
- **group**: fusiona los índices de los miembros (`P:`+`V:` único, gana el primero) y sirve cada
  `.apk` desde el primer miembro que lo tenga.

### Prueba rápida

```bash
BASE=http://localhost:8081/repository/alpine-hosted
curl -u admin:$NEXUS_PASSWORD -X POST http://localhost:8081/service/rest/v1/repositories/alpine/hosted \
  -H 'Content-Type: application/json' -d '{
    "name":"alpine-hosted","online":true,
    "storage":{"blobStoreName":"default","strictContentTypeValidation":false,"writePolicy":"ALLOW"},
    "cleanup":{"policyNames":[]},"component":{"proprietaryComponents":false},"alpine":{}}'
curl -u admin:$NEXUS_PASSWORD -X PUT -H 'Content-Type: application/octet-stream' \
  --data-binary @musl-1.2.5-r4.apk "$BASE/v3.20/main/x86_64/musl-1.2.5-r4.apk"
curl -s "$BASE/v3.20/main/x86_64/APKINDEX.tar.gz" -o /tmp/APKINDEX.tar.gz
apk add --repository "$BASE/v3.20/main" --allow-untrusted musl
```

### Verificación E2E (2026-10-26, sobre la assembly real en `:8081`)

Ejecutado con `/tmp/opencode/e2e_alpine.sh` (fixtures `musl-1.2.5-r4.apk` y
`aaudit-0.7.2-r3.apk` de `v3.20/main/x86_64`, cliente `apk-tools` 2.14.4 estático): **todo en
verde** (26 comprobaciones).

- Recetas `alpine`/`hosted`, `alpine`/`proxy` y `alpine`/`group` presentes en la REST API interna.
- `GET /` en los tres tipos → `200 text/html`; `HEAD` del índice → `200`.
- Repos creados por REST (`201`): `alpine-hosted`, `alpine-proxy`
  (`remoteUrl=https://dl-cdn.alpinelinux.org/alpine`) y `alpine-group`.
- Hosted: `PUT` del `.apk` `201`; payload no APK `400`; `PUT` de `APKINDEX.tar.gz` `400`.
- Índice hosted `200`; la estancia de `musl` es **idéntica byte a byte** a la que produce
  `apk index`; el `.apk` descargado es idéntico al subido.
- Proxy: índice y `.apk` `200`, **byte a byte** iguales al upstream.
- Group: índice fusionado `200` con paquetes del hosted y del proxy, sin duplicar `musl`
  (`P:`+`V:`); sirve el `.apk` del hosted y el del proxy.
- `apk add musl` desde el hosted → **OK** (instala `lib/ld-musl-x86_64.so.1`).
- `apk add busybox` desde el group → **OK**: `busybox` se descarga del proxy y su dependencia
  `musl` del hosted (primer miembro), con instalación `--no-scripts --no-chown` por ejecutarse
  como usuario no root.
- `DELETE` del `.apk` `204`; descarga posterior `404`; el índice del directorio queda vacío
  (tar.gz válido sin estancias).
- `mvn -o -pl public/common/components/formats/nexus-repository-alpine test` → **9 tests, 0
  fallos** (`AlpinePackageMetadataTest` 4, `AlpineIndexBuilderTest` 5).

### Limitaciones

- Solo APK v2; los paquetes APK v3/ADB se rechazan.
- El índice generado no está firmado; los paquetes conservan su propia firma.
- Al subir un `.apk` conviene usar `Content-Type: application/octet-stream` (o cualquier tipo que
  no sea `application/x-www-form-urlencoded`): Jetty intenta parsear los formularios y rechaza
  cuerpos binarios grandes antes de llegar al handler.

## 4. NuGet, 5. Cargo, 6. Conan, 7. CocoaPods, 8. p2, 9. Swift, 10. Git LFS

_(se documentan en cuanto cada módulo queda verde y verificado E2E contra la assembly)_

## 11. Cómo añadir otro formato

Guía operativa para agentes y humanos: `/tmp/opencode/FORMAT_GUIDE.md` (checklist de 6 pasos:
poms → descubrimiento Spring/`@AvailabilityVersion` → REST `@Path` → facet de contenido →
recetas → tests), más el checklist completo de registro generado sobre el árbol real
(build wiring, seguridad, REST, frontend, limpieza).
