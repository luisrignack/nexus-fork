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
| **`ansible`** | hosted + proxy + group | ✅ nuevo + E2E | `nexus-repository-ansible` | 24 |
| `nuget` | hosted + proxy + group | 🚧 en curso | `nexus-repository-nuget` | — |
| `cargo` | hosted + proxy + group | 🚧 en curso | `nexus-repository-cargo` | — |
| `conan` | hosted + proxy + group | 🚧 en curso | `nexus-repository-conan` | — |
| `cocoapods` | proxy | ⏳ pendiente | `nexus-repository-cocoapods` | — |
| `p2` | proxy | ⏳ pendiente | `nexus-repository-p2` | — |
| `swift` | hosted + proxy | ⏳ pendiente | `nexus-repository-swift` | — |
| `gitlfs` | hosted | ⏳ pendiente | `nexus-repository-gitlfs` | — |

También están registrados en `DatastoreHighAvailabilitySupportChecker` (todos los anteriores, incluido
`ansible`) para que la alta disponibilidad los acepte.

## 2. Ansible (Galaxy collections)

Recetas: `ansible-hosted`, `ansible-proxy`, `ansible-group`.

### Endpoints

Bajo la base del repositorio (`http://host:8081/repository/<repo>/`):

| Método | Ruta | Respuesta |
|---|---|---|
| GET | `/api/v3/` | raíz de la API (`collection_index`, `available_versions`) |
| GET | `/api/v3/plugin/ansible/content/published/collections/index/` | todas las collections |
| GET | `…/index/{ns}/` | collections de un namespace |
| GET | `…/index/{ns}/{name}/` | detalle (`highest_version`, `dependencies`, `version_count`) |
| GET | `…/index/{ns}/{name}/versions/` | versiones (la más reciente primero) |
| GET | `…/index/{ns}/{name}/versions/{v}/` | detalle de versión (`artifact`, `manifest`, `download_url`) |
| GET | `/api/v3/collections/{ns}/{name}/{v}/download/` | tarball |
| GET | `/api/v2/collections/{ns}/{name}/[versions[/ {v}]]` | compatibilidad Galaxy v2 |
| PUT | `…/versions/{v}/` (body = tar.gz) o `/collections/{ns}/{name}/{v}/{ns}.{name}-{v}.tar.gz` | publicación (hosted) |
| DELETE | `…/index/{ns}/{name}…` | borra la collection |
| GET | `/` | redirige `302` a `api/v3/` |

### Almacenamiento

```
collections/{namespace}/{name}/{version}/{namespace}.{name}-{version}.tar.gz
```

El `MANIFEST.json` del tarball es la fuente de verdad: si no coincide con las coordenadas de la URL se
responde **400**. Descripción, autores y dependencias se guardan como atributos del componente
(`ansible.description`, `ansible.authors`, `ansible.dependencies`).

### Por tipo

- **hosted**: lee/escribe el almacén y genera la API Galaxy desde los componentes almacenados.
- **proxy**: reenvía a `proxy.remoteUrl`, cachea (metadatos vs. contenido) y reescribe las URLs
  absolutas de las respuestas JSON para que apunten al repositorio local.
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
| `GET /` en hosted, proxy y group | **302** → `api/v3/` |
| `PUT` por `…/index/{ns}/{n}/versions/{v}/` y por `/collections/{ns}/{n}/{v}/{n}.{n}-{v}.tar.gz` | 201 |
| `PUT` sin coordenadas / coordenadas ≠ `MANIFEST.json` / tarball que no es collection | 400 / 400 / 400 |
| Índice, namespace, detalle, versiones y detalle de versión (v3) | 200 (2 versiones, `manifest`, `artifact.sha256`) |
| `GET …/download` | 200 `application/x-gzip`, `gzip -t` OK; versión inexistente → 404 |
| Galaxy v2 (`/api/v2/collections/…`) | 200 |
| Group: índice fusionado con URLs reescritas a `ansible-group` + descarga | 200 + gzip OK |
| Proxy: índice con URLs reescritas a `ansible-proxy` + descarga del tarball | 200 + gzip OK |
| `DELETE …/index/{ns}/{n}` (y segundo DELETE) | 204 / 404 |

### Limitaciones

- Los listados llevan la colección dos veces (`data` para v3 y `results` para v2) desde un único generador.
- `links.next` es siempre `null` (listado completo por página).
- `galaxy.ansible.com` devuelve 403 desde la red de este entorno: la validación del proxy se hizo
  contra un upstream falso local (`/tmp/opencode/fake_galaxy.py`); falta repetirla contra un
  Galaxy real accesible.

## 3. NuGet, 4. Cargo, 5. Conan, 6. CocoaPods, 7. p2, 8. Swift, 9. Git LFS

_(se documentan en cuanto cada módulo queda verde y verificado E2E contra la assembly)_

## 10. Cómo añadir otro formato

Guía operativa para agentes y humanos: `/tmp/opencode/FORMAT_GUIDE.md` (checklist de 6 pasos:
poms → descubrimiento Spring/`@AvailabilityVersion` → REST `@Path` → facet de contenido →
recetas → tests), más el checklist completo de registro generado sobre el árbol real
(build wiring, seguridad, REST, frontend, limpieza).
