# Validación E2E de Helm, Go y RubyGems

> Ejecutada el 2026-10-24 contra la instancia local del fork (`http://localhost:8081`,
> `Nexus/3.89.0-09 (CORE)`, H2 embebido en `sonatype-work`). Herramienta: `curl` sobre la REST API
> (`/service/rest/v1/*`) y los endpoints de contenido (`/repository/<name>/...`).

## 1. Resumen

Se validaron los tres formatos (`helm`, `go`, `rubygems`) en sus tres tipos (hosted, proxy, group)
con artefactos reales: un chart `.tgz` fabricado con `tar`, un módulo Go (`.mod`/`.info`/`.zip`) y
dos `.gem` construidos manualmente, más descargas reales desde `proxy.golang.org` y
`rubygems.org`.

Resultado: **los tres formatos funcionan de extremo a extremo** tras corregir dos defectos de
regresión que se detectaron durante la propia validación (ver §4).

## 2. Repositorios de prueba

Se configuraron los 9 repos + `helm-hosted3` (repo limpio de contraste) y `helm-group2` (grupo
limpio de contraste). Los tres grupos (`helm-group`, `go-group`, `gems-group`) estaban creados sin
`memberNames`; se configuraron vía `PUT /service/rest/v1/repositories/<format>/group/<name>` con
`{"group":{"memberNames":[...]}}`.

| Repositorio | Tipo | Configuración usada |
|---|---|---|
| helm-hosted / helm-hosted3 | hosted | blob store `default` |
| helm-proxy | proxy | remoto `https://charts.bitnami.com/bitnami` (302 → `repo.broadcom.com`) |
| helm-group / helm-group2 | group | miembros `helm-hosted(+3)` y `helm-proxy` |
| go-hosted | hosted | — |
| go-proxy | proxy | remoto `https://proxy.golang.org/` |
| go-group | group | miembros `go-hosted` y `go-proxy` |
| gems-hosted | hosted | — |
| gems-proxy | proxy | remoto `https://rubygems.org/` |
| gems-group | group | miembros `gems-hosted` y `gems-proxy` |

## 3. Matriz de pruebas y resultados

### Helm

| # | Caso | Endpoint | Resultado |
|---|---|---|---|
| H1 | Publicar chart | `PUT /repository/helm-hosted/charts/demo-1.0.0.tgz` | **201** ✅ |
| H2 | Descargar chart | `GET /repository/helm-hosted/charts/demo-1.1.0.tgz` | **200**, md5 idéntico al original ✅ |
| H3 | `index.yaml` (repo limpio) | `GET /repository/helm-hosted3/index.yaml` | **200**, entra `demo 1.2.0` con URL relativa ✅ |
| H4 | `index.yaml` (repo preexistente) | `GET /repository/helm-hosted/index.yaml` | ⚠️ defecto D1 detectado, corregido en §4 |
| H5 | Proxy (índice remoto) | `GET /repository/helm-proxy/index.yaml` | **200**, 27,5 MB del índice Bitnami real ✅ (siguió la 302 de `charts.bitnami.com`) |
| H6 | Group (merge) | `GET /repository/helm-group2/index.yaml` | ⚠️ defecto D2 detectado, corregido en §4 |
| H7 | Componentes en DB | `GET /service/rest/v1/components?repository=helm-hosted` | componente `demo` con versiones y asset `/charts/...tgz` ✅ |

### Go (GOPROXY)

| # | Caso | Endpoint | Resultado |
|---|---|---|---|
| G1 | Publicar módulo | `PUT /repository/go-hosted/example.com/demo/@v/v1.0.0.{mod,info,zip}` | **201** cuando el asset no existía ✅ |
| G2 | Re-publicar | mismo `PUT` sobre assets existentes | ⚠️ defecto D1 (500), corregido en §4 |
| G3 | Lista de versiones hosted | `GET /repository/go-hosted/example.com/demo/@v/list` | **200** → `v1.0.0` ✅ |
| G4 | Descargar `.mod` | `GET /repository/go-hosted/.../@v/v1.0.0.mod` | **200**, 33 bytes, contenido correcto ✅ |
| G5 | Proxy real | `GET /repository/go-proxy/github.com/go-yaml/yaml/@v/list` | **200** con versiones de `proxy.golang.org` ✅ |
| G6 | Proxy descarga zip | `GET /repository/go-proxy/github.com/go-yaml/yaml/@v/v2.0.0+incompatible.zip` | **200**, 72 512 bytes ✅ |
| G7 | Group merge | `GET /repository/go-group/example.com/demo/@v/list` | `v1.0.0` (hosted) ✅ |
| G8 | Group vía proxy | `GET /repository/go-group/github.com/go-yaml/yaml/@v/list` | versiones cacheadas del proxy ✅ |

### RubyGems

| # | Caso | Endpoint | Resultado |
|---|---|---|---|
| R1 | Publicar gem | `PUT /repository/gems-hosted/gems/demo-1.0.0.gem` | **201** ✅ (requiere `Content-Type: application/octet-stream`; sin él, Jetty intenta parseeo de formulario y falla) |
| R2 | Descargar gem | `GET /repository/gems-hosted/gems/demo-1.0.0.gem` | **200**, 10 240 bytes, md5 idéntico ✅ |
| R3 | Componentes | REST `components?repository=gems-hosted` | componentes `demo` 1.0.0 / 2.0.0 con asset `/gems/*.gem` ✅ |
| R4 | Proxy real | `GET /repository/gems-proxy/gems/rack-2.2.8.gem` | **200**, 152 064 bytes, `POSIX tar archive` con `metadata.gz` (verificado con `tar`) ✅ |
| R5 | Proxy `/api/v1/dependencies` | `GET /repository/gems-proxy/api/v1/dependencies?gem_name=rack` | 404 passthrough — **limitación externa**: RubyGems.org retiró esa API en 2023; los clientes modernos usan `specs.4.8.gz` |
| R6 | Índices hosted | `GET /repository/gems-hosted/specs.4.8.gz` | ⚠️ defecto D1 (contenía la copia cacheada vacía), corregido en §4 |

## 4. Defectos detectados y corregidos

### D1 — Re-publicación/regeneración sobre assets existentes falla (`AttributesHelper.overlay`)

**Síntoma.** Cualquier `PUT` que actualice un asset ya existente provisionalmente fallaba:

- `go-hosted`: re-upload del mismo módulo → **500** (`IllegalArgumentException: Conflict: cannot
  overlay 'v1.0.0.mod' onto 'v1.0.0.mod'`).
- `helm`/`rubygems`: los índices generados (`index.yaml`, `specs.4.8.gz`) se cacheaban la primera
  vez que se pedían y la regeneración en cada `PUT` **fallaba en silencio** (el handler traga las
  excepciones), dejando el índice obsoleto (vacío) servida por siempre.

**Causa raíz.** `FluentAssetBuilderImpl.updateAssetAttributes` aplica **siempre** la operación
`AttributeOperation.OVERLAY` a los atributos del asset. `AttributesHelper.overlay` exige que el
valor existente y el nuevo sean ambos `Map` (la semántica de overlay es para mapas anidados tipo
`content.*`), por lo que sobreescribir atributos escalares (p. ej. `go.filename="v1.0.0.mod"`)
lanzaba la excepción. Los formatos del upstream no sufren esto porque sus atributos son mapas
anidados; los formatos propios del fork usan claves planas.

**Corrección.** `public/common/components/nexus-repository-content/.../fluent/internal/FluentAssetBuilderImpl.java`:
usar `SET` para valores no-`Map` y conservar `OVERLAY` para mapas. Semántica del upstream
intacta; verificación: 431 tests de `nexus-repository-content` en verde.

### D2 — Merge de índices en group de Helm no parseaba nada

**Síntoma.** `GET /repository/helm-group(2)/index.yaml` devolvía siempre `entries: {}` aunque los
miembros sirvieran índices válidos.

**Causa raíz.** `HelmIndexBuilder.parseIndex` era un parser de líneas que exigía que la clave del
chart estuviera **sin indentación**, pero el propio builder la escribe con 2 espacios (`  demo:`) y
Bitnami usa el mismo indentado; además confundiría claves anidadas (`deprecated:`, `keywords:`)
con nombres de chart.

**Corrección.** `public/common/components/formats/nexus-repository-helm/.../HelmIndexBuilder.java`:
el merge ahora parsea el YAML de cada miembro con **SnakeYAML** (`org.yaml:snakeyaml`, dependencia
agregada al `pom.xml`, versión 2.3 ya gestionada en el parent y presente en la assembly). Se
elimina el parser manual de líneas.

**Nota** (comportamiento residual de caché): en repos *preexistentes* el índice cacheado vacío
sigue servido hasta el siguiente `PUT` de un chart (la regeneración ocndicional sobreescribe). En
instalaciones nuevas el problema no existe. Los grupos recomputan el merge en cada petición de
`/index.yaml`.

## 5. Limitaciones conocidas (no defectos del fork)

1. `charts.helm.sh/stable` ya no sirve `index.yaml` (archivo removido por el proyecto Helm); se usó
   `charts.bitnami.com/bitnami` (proxy acepta el 302 → `repo.broadcom.com`).
2. `rubygems.org` retiró `/api/v1/dependencies` en 2023 (R5). `gem` moderno funciona vía
   `specs.4.8.gz`.
3. Ruby no está instalado en la máquina de pruebas; se fabricó un `.gem` minimo (`tar` con
   `metadata.gz` + `data.tar.gz`) suficiente para E2E de transporte (los `.gem` reales de
   rubygems.org se descargaron y quedaron maquetados correctamente contra el proxy).

## 6. Cómo repetir la validación

```bash
# $NEXUS_PASSWORD: valor local en nexus-public/admin-creds.txt (ignorado por git)
export ADMIN="admin:$NEXUS_PASSWORD"; BASE=http://localhost:8081

# Helm: publicar y consultar índice
printf 'apiVersion: v2\nname: demo\nversion: 1.0.0\n' > Chart.yaml
printf 'replicaCount: 1\n' > values.yaml
tar czf demo-1.0.0.tgz Chart.yaml values.yaml
curl -u "$ADMIN" -T demo-1.0.0.tgz -X PUT -H 'Content-Type: application/octet-stream' \
  "$BASE/repository/helm-hosted/charts/demo-1.0.0.tgz"
curl "$BASE/repository/helm-hosted/index.yaml"

# Go: publicar y listar
curl -u "$ADMIN" -H 'Content-Type: text/plain' --data-binary 'module example.com/demo
go 1.21' -X PUT "$BASE/repository/go-hosted/example.com/demo/@v/v1.0.0.mod"
curl "$BASE/repository/go-hosted/example.com/demo/@v/list"

# RubyGems: publicar y descargar
curl -u "$ADMIN" -H 'Content-Type: application/octet-stream' --data-binary @demo-1.0.0.gem \
  -X PUT "$BASE/repository/gems-hosted/gems/demo-1.0.0.gem"
curl "$BASE/repository/gems-hosted/gems/demo-1.0.0.gem" -o /tmp/down.gem && md5sum /tmp/down.gem

# Proxies reales
curl "$BASE/repository/go-proxy/github.com/go-yaml/yaml/@v/list"
curl "$BASE/repository/helm-proxy/index.yaml" -o /tmp/bitnami-index.yaml   # ~27 MB
curl "$BASE/repository/gems-proxy/gems/rack-2.2.8.gem" -o /tmp/rack.gem
```

## 7. Pendiente posterior

- NuGet sigue sin implementar (paso 3 del plan general).
- Validar `helm-proxy` contra un remoto que sirva tarballs para cachear también los `.tgz`
  (el índice Bitnami ya se cachea, verificado en H5).
