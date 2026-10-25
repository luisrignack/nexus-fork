# Proxy Docker / OCI — guía de uso y diseño

> **Credenciales:** los ejemplos usan `admin:$NEXUS_PASSWORD`. Define la variable antes de
> ejecutarlos (`export NEXUS_PASSWORD='...'`; el valor local está en
> `nexus-public/admin-creds.txt`, ignorado por git) o sustitúyelo por tu usuario y contraseña.
>
> Estado: **funcional y verificado** (`ghcr` y `quay` de extremo a extremo; `dockerhub` y `gcr`
> con limitación de red, ver §9).
> Módulo: `nexus-public/public/common/components/formats/nexus-repository-docker/`
> Licencia: EPL-1.0.

## 1. Qué es

Un **proxy pull-through** del API Registry v2 montado directamente en `/v2/`, sin el prefijo
`/repository/<nombre>` que usan los demás formatos. Los clientes Docker/OCI apuntan al puerto de
Nexus y este descarga manifests, listas de tags y blobs de uno de los mirrors configurados,
guardándolos en disco.

No es un registro hosted: **no implementa push**. Los mirrors de Daocloud son de solo lectura y el
objetivo es caché/aceleración, no publicar imágenes.

```
docker ──► nexus:8081/v2/... ──► mirror (daocloud) ──► registro real
                    │
                    └── caché en disco (manifests, tags, blobs)
```

## 2. Repositorios Docker creados desde la UI

Además del selector global de mirrors, este fork registra el formato `docker` con la receta
`docker-proxy`, de modo que **Administration → Repositories → Create repository** ofrece
`docker` / `proxy` como cualquier otro formato. La lista de recetas que consume la UI es
`GET /service/rest/internal/ui/repositories/recipes` y devuelve:

```json
{"format": "docker", "type": "proxy"}
```

La receta adjunta `ConfigurableViewFacet`, `SecurityFacet`, `HttpClientFacet` y
`DockerProxyFacet`; no crea conectores Jetty por repositorio. El propio formulario explica la
ruta de acceso (`Registry Access`):

```
docker pull <host>:<puerto>/<nombre-repo>/<imagen>:<tag>
```

El upstream del repositorio se toma de `dockerProxy.indexUrl` y, si está vacío, de
`proxy.remoteUrl`. Con `indexType=HUB` se usa el mirror global `dockerhub`; con
`indexType=REGISTRY` (por defecto) se usa la URL del proxy; con `CUSTOM` se usa la URL indicada.
Las credenciales Basic/Bearer de `httpclient.authentication` se descifran con `SecretsService` y
se envían al registro.

### Crear por REST

```bash
curl -u admin:$NEXUS_PASSWORD -H 'Content-Type: application/json' \
  -X POST http://localhost:8081/service/rest/v1/repositories/docker/proxy -d '{
  "name": "docker-ghcr",
  "online": true,
  "storage": {"blobStoreName": "default", "strictContentTypeValidation": true, "writePolicy": "ALLOW"},
  "proxy": {"remoteUrl": "https://ghcr.io", "contentMaxAge": 1440, "metadataMaxAge": 1440},
  "negativeCache": {"enabled": true, "timeToLive": 1440},
  "httpClient": {"blocked": false, "autoBlock": true,
    "connection": {"retries": 0, "userAgentSuffix": "", "timeout": 60,
                   "enableCircularRedirects": false, "enableCookies": false},
    "authentication": null},
  "routingRule": null,
  "docker": {"httpPort": null, "httpsPort": null, "forceBasicAuth": false,
             "v1Enabled": false, "subdomain": null},
  "dockerProxy": {"indexType": "REGISTRY", "indexUrl": "https://ghcr.io",
                  "cacheForeignLayers": false, "foreignLayerUrlWhitelist": []}
}'
```

También existen los endpoints V1 y Beta equivalentes:

```
POST/PUT/GET /service/rest/v1/repositories/docker/proxy[/{name}]
POST/PUT/GET /service/rest/beta/repositories/docker/proxy[/{name}]
```

### Pull contra un repositorio

```bash
docker pull localhost:8081/docker-ghcr/astral-sh/uv:latest
```

El primer segmento tras `/v2/` (`docker-ghcr`) selecciona el repositorio y el resto
(`astral-sh/uv`) se reenvía al upstream configurado. El repositorio no usa el blob store de
Nexus: los manifests y blobs se guardan en la caché en disco descrita en §7.

## 3. Selección de upstream por ruta

El **primer segmento** de la ruta decide el destino. Si coincide con el nombre de un repositorio
de formato `docker`, se usa el upstream de ese repositorio. Si no, se aplica el selector global
de mirrors que se describe a continuación.

| Ruta en Nexus | Upstream | Registro real |
|---|---|---|
| `/v2/library/nginx/...` | `dockerhub` (por defecto) | Docker Hub |
| `/v2/dockerhub/library/nginx/...` | `dockerhub` | Docker Hub |
| `/v2/gcr/distroless/static/...` | `gcr` | Google Container Registry |
| `/v2/ghcr/owner/image/...` | `ghcr` | GitHub Container Registry |
| `/v2/quay/prometheus/node-exporter/...` | `quay` | Quay.io |

- Si el primer segmento **no** coincide con ningún upstream configurado, se usa el upstream por
  defecto y se reenvía la ruta completa. Así `docker pull localhost:8081/nginx` funciona como en
  Docker Hub.
- Si el primer segmento coincide pero no queda ruta después (`/v2/gcr`), también se trata como
  ruta normal del upstream por defecto.
- **Fallback `library/`**: en Docker Hub, una imagen sin `/` (`nginx`) se pide como `library/nginx`.
  Se puede desactivar con `nexus.docker.hubLibraryFallback=false`.

## 4. Configuración

Todas las propiedades se leen del entorno de Spring (`nexus.properties`, variables de entorno o
`-D`). Con la assembly de trabajo el archivo es:

```
.../target/sonatype-work/nexus3/etc/nexus.properties
```

| Propiedad | Por defecto | Descripción |
|---|---|---|
| `nexus.docker.enabled` | `true` | Si es `false`, `/v2/` responde 404. |
| `nexus.docker.upstreams` | `dockerhub=https://docker.m.daocloud.io,gcr=https://gcr.m.daocloud.io,ghcr=https://ghcr.m.daocloud.io,quay=https://quay.m.daocloud.io` | Lista `nombre=url` separada por comas. |
| `nexus.docker.defaultUpstream` | `dockerhub` | Upstream usado cuando la ruta no empieza por un nombre conocido. |
| `nexus.docker.cacheDir` | `${karaf.data}/docker-cache` | Directorio de caché. |
| `nexus.docker.manifestTtlSeconds` | `300` | TTL de manifests pedidos **por tag**. Los pedidos por digest son inmutables. |
| `nexus.docker.tagsTtlSeconds` | `300` | TTL de `/tags/list`. |
| `nexus.docker.timeoutSeconds` | `60` | Timeout de conexión/lectura contra el upstream. |
| `nexus.docker.hubLibraryFallback` | `true` | Añade `library/` a imágenes oficiales de Docker Hub sin `/`. |

Ejemplo para cambiar de mirror sin recompilar:

```properties
nexus.docker.upstreams=dockerhub=https://docker.m.daocloud.io,ghcr=https://ghcr.m.daocloud.io
nexus.docker.defaultUpstream=dockerhub
nexus.docker.manifestTtlSeconds=600
nexus.docker.tagsTtlSeconds=600
```

### Credenciales de un upstream

Se pueden incrustar en la URL:

```properties
nexus.docker.upstreams=private=https://user:secret@registry.example.com
```

`DockerRegistryConfig` separa el `userinfo`, lo convierte en `Authorization: Basic ...` y deja la
URL limpia. **Cuidado:** la propiedad queda en texto plano en `nexus.properties`; para producción
conviene un mirror interno sin credenciales o variables de entorno protegidas.

## 5. Autenticación y autorización

El servlet vive fuera de `/repository/**`, así que el módulo registra sus propias cadenas Shiro
para `/v2` y `/v2/**` (`DockerConfiguration`):

- `nx-authc` procesa `Authorization: Basic` y API keys.
- `nx-anonymous` aplica la configuración de acceso anónimo de Nexus.
- `DockerRegistryService` decide si la petición puede continuar:
  - usuario autenticado (no anónimo) → permitido;
  - acceso anónimo habilitado → permitido;
  - en caso contrario → **401** con `WWW-Authenticate: BASIC realm="Sonatype Nexus Repository Manager"`.

Con el acceso anónimo **deshabilitado**:

```bash
docker login localhost:8081 -u admin
docker pull localhost:8081/quay/prometheus/node-exporter:v1.7.0
```

Con el acceso anónimo **habilitado** no hace falta login para hacer pull.

> Nota de alcance: cualquier usuario autenticado puede hacer pull. Aunque cada repositorio Docker
> registra sus privilegios de formato (`nexus:repository-admin:docker:<nombre>`), el servlet `/v2`
> todavía no evalúa privilegios por repositorio; solo distingue autenticado frente a anónimo.

La autenticación contra el upstream es transparente: si el mirror responde `401` con
`WWW-Authenticate: Bearer`, el proxy pide el token al `realm`, lo cachea hasta su expiración y
reintenta la petición.

## 6. Endpoints soportados

| Método | Ruta | Comportamiento |
|---|---|---|
| `GET`/`HEAD` | `/v2/` | `200 {}` si está autorizado; usado por `docker login`. |
| `GET`/`HEAD` | `/v2/<upstream>/<repo>/manifests/<tag\|digest>` | Manifest o manifest list, con `Accept` reenviado. |
| `GET`/`HEAD` | `/v2/<upstream>/<repo>/blobs/<digest>` | Blob; streaming y caché. |
| `GET`/`HEAD` | `/v2/<upstream>/<repo>/tags/list[?n=&last=]` | Lista de tags. Las listas paginadas no sustituyen la caché completa. |
| otros | cualquier ruta | `405` con `Allow: GET, HEAD`. |

Siempre se devuelve `Docker-Distribution-Api-Version: registry/2.0`.

## 7. Caché

```
<cacheDir>/<upstream>/blobs/<sha256_...>
<cacheDir>/<upstream>/manifests/by-digest/<sha256_...>
<cacheDir>/<upstream>/manifests/by-tag/<repo>/<tag>
<cacheDir>/<upstream>/tags/<repo>/list
```

Cada entrada tiene un sidecar `.meta` con `Content-Type`, `Docker-Content-Digest`, `ETag` y la
fecha de caché.

- **Blobs y manifests por digest**: inmutables, se sirven siempre desde disco tras la primera
  descarga.
- **Manifests por tag y listas de tags**: se refrescan cuando vence el TTL. Si el upstream falla
  con 5xx, se sirve la copia vieja (stale) para no romper pulls.
- **Streaming**: en la primera descarga de un blob se envían las cabeceras y los bytes al cliente
  mientras se escribe el archivo temporal; al terminar se mueve a la caché. Si el cliente corta,
  la descarga continúa para completar la entrada.
- **Range**: se atiende desde caché con `206` y `Content-Range`. Un `Range` contra un blob que aún
  no está cacheado se reenvía al upstream y **no** se cachea parcialmente.
- **ETag / If-None-Match**: soportado en las respuestas servidas desde caché (`304`).
- **Garbage Collection (`repository.docker-gc`)**: tarea programable (*Docker - Garbage collect
  pull-through cache*) con soporte para:
  - `maxSizeMb`: tamaño máximo en MB por upstream (evicción LRU según `cachedAt` del `.meta`).
  - `maxAgeHours`: edad máxima en horas de las entradas.
  - `upstream`: filtro opcional de upstream o repositorio (vacío = todos).
  - `dryRun`: calcula y registra en el log qué entradas se borrarían sin eliminarlas del disco.
  - **Protección de referencias**: los blobs y manifests por digest referenciados por manifests
    activos (`config.digest`, `layers[].digest`, `manifests[].digest`) quedan protegidos frente al
    borrado.

## 8. Pruebas rápidas

```bash
# ping (401 si el anónimo está deshabilitado)
curl -i http://localhost:8081/v2/

# manifest de una imagen multi-arquitectura
ACC='application/vnd.oci.image.index.v1+json,application/vnd.docker.distribution.manifest.list.v2+json,application/vnd.docker.distribution.manifest.v2+json,application/vnd.oci.image.manifest.v1+json'
curl -u admin:$NEXUS_PASSWORD -H "Accept: $ACC" \
  http://localhost:8081/v2/quay/prometheus/node-exporter/manifests/v1.7.0

# tags
curl -u admin:$NEXUS_PASSWORD 'http://localhost:8081/v2/quay/prometheus/node-exporter/tags/list?n=10'

# pull real
docker pull localhost:8081/ghcr/stefanprodan/podinfo:6.0.0
docker pull localhost:8081/quay/prometheus/node-exporter:v1.7.0

# segunda descarga: debe ser inmediata (caché)
curl -o /dev/null -w '%{time_total}\n' -u admin:$NEXUS_PASSWORD \
  http://localhost:8081/v2/quay/prometheus/node-exporter/blobs/<digest>
```

Tests unitarios del módulo:

```bash
cd nexus-public
mvn -o -pl public/common/components/formats/nexus-repository-docker test
```

Cubren parseo de upstreams, resolución de prefijos, fallback `library/`, rutas de caché, TTL,
sanitización de nombres, `Bearer` challenge, `Range`, digests, el facet de repositorio y
garbage collection de la caché (31 tests).

## 9. Limitaciones conocidas

1. **Daocloud + Docker Hub (blobs)**: los manifests se descargan bien, pero los blobs redirigen a
   `production.cloudfront.docker.com`, que responde **403 por geobloqueo** desde esta red.
   `docker pull` de imágenes de Docker Hub falla al bajar las capas.
2. **Daocloud + GCR (blobs)**: igual, los blobs redirigen a Google Storage y devuelven **403** en
   esta red. Los manifests de `gcr.m.daocloud.io` sí funcionan.
3. **GHCR y Quay**: funcionan completos (manifests, tags, blobs y `docker pull`) a través de
   `ghcr.m.daocloud.io` y `quay.m.daocloud.io`.
4. **Sin push**: los mirrors son read-only y el módulo no implementa el flujo de upload.
5. **Un solo nivel de caché**: no hay compartición entre nodos (ver HA en
   `docs/FUNCIONALIDADES_PRO.md`).
6. **RBAC por repositorio/upstream**: no implementado en el servlet `/v2` (ver §5).

Para Docker Hub/GCR en una red sin geobloqueo, o con otro mirror, basta cambiar
`nexus.docker.upstreams`. El código no está atado a Daocloud.

## 10. Clases principales

| Clase | Responsabilidad |
|---|---|
| `DockerRegistryServlet` | `@WebServlet("/v2", "/v2/*")`; punto de entrada Jetty. |
| `DockerRegistryService` | Proxy Registry v2: routing, tokens Bearer, descarga, streaming, caché y `Range`. |
| `DockerRegistryConfig` | Upstreams, resolución de prefijos, TTLs, fallback `library/`. |
| `DockerCache` | Caché en disco con sidecars `.meta`. |
| `DockerConfiguration` | Cadenas Shiro para `/v2` y `/v2/**`. |
| `DockerFormat` | Formato `docker` registrado en Nexus. |
| `DockerProxyRecipe` | Receta `docker-proxy` que hace visible el formato en la UI y adjunta los facets. |
| `DockerProxyFacet` / `DockerProxyFacetImpl` | Resuelve `dockerProxy.indexUrl`/`proxy.remoteUrl` y las credenciales del repositorio. |
| `DockerStoresProvider` / `DockerStoresConfiguration` | Stores de contenido del formato (requeridos por el registro de formatos). |
| `DockerProxyRepositoriesApiResourceV1` / `...Beta` | REST de creación/edición de repositorios Docker. |
| `DockerRepositoryAdapter` | Adapta el repositorio al modelo REST V1/Beta. |
