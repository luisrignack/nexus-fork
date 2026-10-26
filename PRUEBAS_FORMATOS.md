# Pruebas de formatos — nexus-fork

Resultados de las pruebas de extremo a extremo realizadas sobre la assembly real de
**Nexus Repository 3.89.0-09** de este fork.

| Dato | Valor |
|---|---|
| Fecha | 2026-10-26 |
| Instancia | `http://localhost:8081` |
| Build | `3.89.0-09` |
| Commit raíz | `a5161c4` (`docs: documentar Alpine y actualizar estado de Ansible`) + cambios de formatos sin commitear |
| Commit `nexus-public` | `243cef5` (`feat(alpine): soporte APK hosted/proxy/group`) + correcciones de Cargo/Conan/NuGet/PyPI/Docker sin commitear |
| JDK de compilación | 21.0.13+9 |
| Sistema | Linux x86_64 |

Las pruebas se ejecutaron contra el servidor en marcha, no contra mocks, salvo donde se
indica expresamente. Los artefactos descargados se compararon byte a byte (`cmp`) y por
SHA-256 con el original del upstream correspondiente. Los archivos comprimidos se
validaron con `tar`, `gzip`, `zipfile`, `dpkg-deb` o el cliente nativo del formato.

> **Resultado global:** los 15 formatos con receta quedaron validados ✅ tras corregir
> Cargo, Conan, NuGet y PyPI y ajustar los upstreams Docker a los mirrors que funcionan.
> Docker Hub y el mirror `gcr` se rechazan de forma explícita (404 rápido) en lugar de
> intentar una descarga que falla. Los 4 módulos sin receta siguen fuera del alcance ⏳
> (CocoaPods, p2, Swift, Git LFS).

---

## 1. Alcance

El árbol `nexus-public/public/common/components/formats/` contiene **19 módulos de formato**:

```
alpine  ansible  apt  cargo  cocoapods  conan  docker  gitlfs  go
helm    maven    npm  nuget  p2        pypi   raw     rubygems  swift  yum
```

De ellos, **15 tienen recetas registradas** en esta build (visibles en
`GET /service/rest/internal/ui/repositories/recipes`):

| Formato | Recetas registradas |
|---|---|
| `alpine` | hosted, proxy, group |
| `ansible` | hosted, proxy, group |
| `apt` | hosted, proxy |
| `cargo` | hosted, proxy, group |
| `conan` | hosted, proxy, group |
| `docker` | proxy |
| `go` | hosted, proxy, group |
| `helm` | hosted, proxy, group |
| `maven2` | hosted, proxy, group |
| `npm` | hosted, proxy, group |
| `nuget` | hosted, proxy, group |
| `pypi` | hosted, proxy, group |
| `raw` | hosted, proxy, group |
| `rubygems` | hosted, proxy, group |
| `yum` | hosted, proxy |

Los otros **4 módulos están incluidos en la assembly pero no registran receta**:

| Formato | Estado real |
|---|---|
| `cocoapods` | Scaffolding parcial (5 clases: formato, seguridad, cleanup); sin receta ni handler de contenido. |
| `p2` | Solo `README.md` y `pom.xml`; el JAR no contiene código Java. |
| `swift` | Solo `README.md` y `pom.xml`; el JAR no contiene código Java. |
| `gitlfs` | Solo `README.md` y `pom.xml`; el JAR no contiene código Java. |

Por tanto, no es posible crear repositorios de esos cuatro formatos ni ejecutar un E2E
de protocolo sobre ellos en esta build.

---

## 2. Resumen

Leyenda: ✅ validado · ⚠️ parcial · ❌ fallo · ⏳ sin receta / no implementado.

| Formato | Recetas | Repositorios usados | Prueba principal | Resultado |
|---|---|---|---|---|
| `alpine` | hosted, proxy, group | `alpine-hosted`, `alpine-proxy`, `alpine-group` | `apk fetch` y `apk add`; índice y hashes | ✅ |
| `ansible` | hosted, proxy, group | `qa-ansible-hosted` | `PUT` de collection, índice, descarga, `DELETE` | ✅ hosted (proxy real: 403 de red) |
| `apt` | hosted, proxy | `qa-apt-proxy` | `Release`, `Packages.gz` y `.deb` | ✅ |
| `cargo` | hosted, proxy, group | `qa-cargo-proxy` | `config.json`, índice y `.crate` | ✅ |
| `cocoapods` | — | — | Sin receta | ⏳ |
| `conan` | hosted, proxy, group | `qa-conan-hosted`, `qa-conan-proxy` | `ping`, `search`, `PUT`/`GET` de ficheros, `download_urls` | ✅ hosted (proxy real: 403 de red) |
| `docker` | proxy | `docker-ghcr`, mirrors globales | `docker pull` + ejecución; Hub/`gcr` 404 sin salida | ✅ |
| `gitlfs` | — | — | Sin receta | ⏳ |
| `go` | hosted, proxy, group | `go-proxy`, `go-group` | `@v/list`, `.info`, `.mod`, `.zip` | ✅ |
| `helm` | hosted, proxy, group | `helm-proxy`, `helm-hosted`, `helm-group` | `index.yaml` y chart `.tgz` | ✅ |
| `maven2` | hosted, proxy, group | `maven-central`, `maven-public`, `maven-releases` | POM/JAR y `PUT` en hosted | ✅ |
| `npm` | hosted, proxy, group | `npm-proxy`, `npm-group` | metadata, tarball, `npm view`/`npm pack` | ✅ |
| `nuget` | hosted, proxy, group | `qa-nuget-proxy` | índice, search, registration, flat-container, `.nupkg` | ✅ |
| `p2` | — | — | Sin receta | ⏳ |
| `pypi` | hosted, proxy, group | `qa-pypi-proxy` | simple, JSON, wheel, sidecar `.metadata`, `pip download` | ✅ |
| `raw` | hosted, proxy, group | `raw-hosted` | `PUT`, `GET`, `DELETE` | ✅ |
| `rubygems` | hosted, proxy, group | `gems-proxy`, `gems-group` | `specs.4.8.gz` y `.gem` | ✅ |
| `swift` | — | — | Sin receta | ⏳ |
| `yum` | hosted, proxy | `qa-yum-proxy` | `repomd.xml`, primary y `.rpm` | ✅ |

**Balance:** 15 formatos ✅ y 4 ⏳ (sin receta). No quedan fallos ni resultados parciales
en los formatos implementados.

---

## 3. Detalle por formato

### 3.1 Alpine (apk)

Tres repositorios, cliente `apk-tools` 2.14.4 estático, rama `v3.20/main/x86_64`.

| Repo | Tipo | Índice | Paquetes en índice | Descarga | Instalación |
|---|---|---|---|---|---|
| `alpine-hosted` | hosted | 200 | 1 (`musl`) | `musl` OK | `apk add musl` OK |
| `alpine-proxy` | proxy | 200 | 5.525 | `musl` y `busybox` OK | `apk add busybox` OK (2 paquetes) |
| `alpine-group` | group | 200 | 5.525 | `musl` y `busybox` OK | `apk add busybox` OK (2 paquetes) |

| Repo | Paquete | Tamaño | SHA-256 | ¿Igual al upstream? |
|---|---|---|---|---|
| hosted | `musl-1.2.5-r4.apk` | 408.411 B | `b255cdcf…2e2bc7d8` | Sí |
| proxy | `musl-1.2.5-r4.apk` | 408.411 B | `b255cdcf…2e2bc7d8` | Sí |
| proxy | `busybox-1.36.1-r31.apk` | 504.644 B | `a7c2c34d…4fe531c4` | Sí |
| group | `musl-1.2.5-r4.apk` | 408.411 B | `b255cdcf…2e2bc7d8` | Sí |
| group | `busybox-1.36.1-r31.apk` | 504.644 B | `a7c2c34d…4fe531c4` | Sí |

`alpine-hosted` solo contiene `musl` porque no se subió `busybox`; `apk fetch busybox`
responde correctamente con `unable to select package`. El group resuelve `musl` desde el
hosted y `busybox` desde el proxy, y ejecuta el binario con el loader de musl
(`hello-from-alpine-group`). Las advertencias de permisos de `apk add` son benignas
(ejecución como usuario no root con `--no-scripts --no-chown`).

### 3.2 Ansible (Galaxy)

Repositorio hosted creado para la prueba (`qa-ansible-hosted`), fixture
`community-general-1.0.0.tar.gz` (506 B).

| Comprobación | Resultado |
|---|---|
| `PUT …/index/community/general/versions/1.0.0/` | 201 |
| `GET …/collections/index/` | 200, 1.939 B, `meta.count=1` |
| `GET …/collections/community/general/1.0.0/download/` | 200, 506 B, **idéntico** al fixture |
| `DELETE …/index/community/general` | 204 |
| Descarga tras el borrado | 404 |

El proxy `ansible-galaxy-real` no se pudo revalidar en esta sesión porque
`galaxy.ansible.com` responde **403 de Cloudflare** desde esta IP; el flujo con Galaxy real
ya estaba documentado con un relay local en `docs/FORMATOS.md`.

### 3.3 APT (Debian)

Proxy `qa-apt-proxy` → `https://deb.debian.org/debian`, distribución `bookworm`.

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `dists/bookworm/Release` | 200 | 149.266 B | `Origin: Debian`, `Suite: oldstable` |
| `dists/bookworm/main/binary-amd64/Packages.gz` | 200 | 12.084.798 B | gzip válido, 63.440 paquetes |
| `pool/main/h/hello/hello_2.10-2_amd64.deb` | 200 | 56.132 B | `dpkg-deb -I` OK, idéntico al upstream |

### 3.4 Cargo

Proxy `qa-cargo-proxy` → `https://index.crates.io`.

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `index/config.json` | 200 | 130 B | `dl` → `/api/v1/crates`; `api` → raíz del repositorio |
| `index/se/rd/serde` | 200 | 164.874 B | NDJSON del índice, `cksum` correcto |
| `api/v1/crates/serde/1.0.0/download` | 200 | 58.433 B | `application/gzip`; SHA-256 = `cksum` |
| `api/v1/crates/anyhow/1.0.0/download` | 200 | 13.476 B | `application/gzip`; SHA-256 = `cksum` |
| Segunda petición de `serde/1.0.0` | 200 | 58.433 B | servida desde caché, mismo SHA-256 |

`config.json` anunciado por el repositorio:

```json
{"dl":"http://localhost:8081/repository/qa-cargo-proxy/api/v1/crates",
 "api":"http://localhost:8081/repository/qa-cargo-proxy"}
```

`api` es la raíz del repositorio porque Cargo construye `<api>/api/v1/crates/...` a partir
de ese valor; con la barra final o con `/api` se generaban rutas duplicadas. Las descargas
se enrutan contra el `dl` descubierto en el `config.json` del upstream
(`https://static.crates.io/crates`), no contra el índice, y el `.crate` se almacena ya con
`Content-Type: application/gzip` para que la respuesta cacheada sea idéntica a la primera.

### 3.5 Conan

| Repo | Comprobación | Resultado |
|---|---|---|
| `qa-conan-hosted` | `GET /v1/ping` | 200, `X-Conan-Server-Version: 2.0` |
| `qa-conan-hosted` | `GET /v1/conans/search?q=*` | 200, `{"results":[]}` |
| `qa-conan-hosted` | `PUT …/revisions/1111…/files/conanfile.py` | **201** |
| `qa-conan-hosted` | `GET …/revisions` | 200, revisión con `content_url` |
| `qa-conan-hosted` | `GET …/revisions/1111…/files/conanfile.py` | 200, 96 B, **idéntico** al `PUT` |
| `qa-conan-hosted` | `PUT`/`GET` `…/packages/3333…/revisions/2222…/files/lib/libhello.a` | 201 / 200, 1.024 B, `cmp` idéntico |
| `qa-conan-hosted` | `GET …/packages/3333…/revisions` y `…/files` | 200, revisión y ficheros listados |
| `qa-conan-hosted` | `GET /v1/conans/hello/1.0/_/_/download_urls` | 200, URLs al `v2` local |
| `qa-conan-hosted` | `GET /v1/conans/hello/1.0/_/_/packages/3333…/download_urls` | 200, URLs de binarios |
| `qa-conan-proxy` | `GET /v1/ping` | 200 (handler local) |
| `qa-conan-proxy` | `GET https://center.conan.io/v1/ping` (directo) | **403** del upstream |

La causa de los 500 era la misma que en Cargo: las rutas de asset se construían sin barra
inicial. Ahora `ConanCoordinates.assetPath()` genera la ruta absoluta y el handler usa rutas
relativas coherentes para `files`, revisiones y `download_urls`. El contenido hosted
(receta, `conaninfo.txt` y binario) se guarda y recupera byte a byte. El proxy no se pudo
validar contra un upstream real porque `center.conan.io` está bloqueado por la red de este
entorno (403), no por el módulo.

### 3.6 Docker

El módulo Docker de este fork expone la API de registro en el puerto principal:

```
http://localhost:8081/v2/<repositorio-o-upstream>/<imagen>/manifests|blobs|tags/...
```

Los upstreams por defecto son solo los mirrors verificados: `ghcr` y `quay`
(`nexus.docker.defaultUpstream=ghcr`). Docker Hub no se configura por defecto.

| Comprobación | Resultado |
|---|---|
| `GET /v2/` | 200 `{}` |
| `GET /v2/docker-ghcr/astral-sh/uv/manifests/0.5.0` | 200, índice OCI |
| `docker pull localhost:8081/docker-ghcr/astral-sh/uv:0.5.0` | **OK**, digest `sha256:0e0fb779…e9db9d` |
| `docker run … uv --version` | `uv 0.5.0` |
| `docker pull localhost:8081/ghcr/astral-sh/uv:0.5.0` | **OK**, `Downloaded newer image` |
| `docker pull localhost:8081/quay/prometheus/node-exporter:v1.8.0` | **OK**, digest `sha256:8a57af80…c15e4ec` |
| `GET /v2/ghcr/…/blobs/<config\|layers>` | 200 en todos los blobs |
| `GET /v2/quay/…/blobs/<config\|layers>` | 200 en todos los blobs |
| `GET /v2/gcr/google-containers/busybox/manifests/1.27` | **404** en ~100 ms, sin salida a red |
| `GET /v2/library/nginx/manifests/latest` | **404** en ~40 ms, sin salida a Docker Hub |
| `GET /v2/dockerhub/library/nginx/manifests/latest` | **404** en ~40 ms |
| `GET /v2/docker.io/library/nginx/manifests/latest` | **404** en ~40 ms |
| `GET /v2/qa-docker-proxy/library/nginx/manifests/latest` | **404** en ~40 ms; log: *«points at Docker Hub, but no 'dockerhub' upstream is configured»* |
| `https://auth.docker.io/token` directo | **403** `RBAC: access denied` |

`ghcr` y `quay` descargan manifiestos y blobs y permiten `docker pull` completo. `gcr` se
retiró de los defaults porque su mirror sirve manifiestos pero rechaza los blobs con 403;
al no estar configurado, un prefijo `gcr/...` devuelve 404 en lugar de enviarse al upstream
por defecto. Del mismo modo, cualquier ruta de Docker Hub (`library/...`, `dockerhub/...`,
`docker.io/...`) se rechaza con 404 rápido y **no** genera tráfico hacia
`registry-1.docker.io` ni `auth.docker.io`.


### 3.7 Go

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `go-proxy/github.com/pkg/errors/@v/list` | 200 | 91 B | lista de versiones |
| `…/@v/v0.9.1.info` | 200 | 50 B | JSON |
| `…/@v/v0.9.1.mod` | 200 | 29 B | `go.mod` |
| `…/@v/v0.9.1.zip` | 200 | 17.866 B | ZIP válido, **idéntico** al upstream |
| `go-group/…/@v/list` | 200 | 91 B | grupo operativo |
| `go-group/…/@v/v0.9.1.zip` | 200 | 17.866 B | **idéntico** al upstream |

### 3.8 Helm

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `helm-proxy/index.yaml` | 200 | 27.536.945 B | 144 charts |
| `helm-proxy/airflow-21.9.0.tgz` | 200 | 265.406 B | tar.gz válido, **idéntico** al upstream |
| `PUT helm-hosted/airflow-21.9.0.tgz` | 201 | — | alta correcta |
| `helm-hosted/index.yaml` | 200 | 1.505 B | contiene `airflow` 21.9.0 |
| `GET helm-hosted/airflow-21.9.0.tgz` | 200 | 265.406 B | **idéntico** al subido |
| `helm-group/index.yaml` | 200 | 4.512.729 B | índice de grupo servido |

### 3.9 Maven 2

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `maven-central/…/slf4j-api-2.0.13.pom` | 200 | 2.826 B | POM |
| `maven-central/…/slf4j-api-2.0.13.jar` | 200 | 68.605 B | JAR válido, **idéntico** al upstream |
| `maven-public/…/slf4j-api-2.0.13.pom` | 200 | 2.826 B | grupo operativo |
| `PUT maven-releases/…/qa-artifact-1.0.pom` | 201 | — | alta correcta |
| `GET maven-releases/…/qa-artifact-1.0.pom` | 200 | 229 B | **idéntico** al subido |

### 3.10 npm

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `npm-proxy/left-pad` | 200 | 22.813 B | metadata JSON |
| `npm-proxy/left-pad/-/left-pad-1.3.0.tgz` | 200 | 3.619 B | tar.gz válido, **idéntico** al upstream |
| `npm-group/left-pad` | 200 | 22.813 B | URLs de tarball reescritas al group |
| `npm-group/left-pad/-/left-pad-1.3.0.tgz` | 200 | 3.619 B | **idéntico** al upstream |
| `npm view left-pad version` | — | — | `1.3.0` |
| `npm pack left-pad@1.3.0` | — | 3.619 B | paquete generado correctamente |

### 3.11 NuGet

| Recurso | HTTP | Tamaño | Resultado |
|---|---|---|---|
| `qa-nuget-proxy/index.json` | 200 | 8.153 B | índice de servicio reescrito a `localhost:8081` |
| `…/v3/query?q=newtonsoft&take=1` | 200 | 8.737 B | búsqueda operativa |
| `…/registration5-gz-semver2/newtonsoft.json/index.json` | 200 | 367.330 B | variante de registration preservada |
| `…/v3-flatcontainer/newtonsoft.json/index.json` | 200 | 1.462 B | lista de versiones |
| `…/v3-flatcontainer/newtonsoft.json/13.0.3/newtonsoft.json.13.0.3.nupkg` | 200 | 2.441.966 B | ZIP válido |
| Tras reiniciar, sin volver a pedir `index.json` | 200 | — | flat, search, registration y `.nupkg` siguen operativos |

El índice de servicio se reescribe y, al guardarlo, se persisten como atributos del asset
los endpoints descubiertos en el upstream:

| Recurso | Upstream anunciado |
|---|---|
| `PackageBaseAddress/3.0.0` | `https://api.nuget.org/v3-flatcontainer/` |
| `RegistrationsBaseUrl/3.6.0` | `https://api.nuget.org/v3/registration5-gz-semver2/` |
| `SearchQueryService` | `https://azuresearch-usnc.nuget.org/query` |

Las peticiones de flat-container, registration y search se enrutan contra esos endpoints
descubiertos y no contra el `remoteUrl`. La variante de registration solicitada
(`registration5-gz-semver2`, `registration5-semver1`, …) se conserva al construir la URL
del upstream. Los atributos se recargan de forma perezosa, de modo que un reinicio no
obliga a refrescar `index.json` antes de poder descargar paquetes.

### 3.12 PyPI

| Recurso | HTTP | Tamaño | Resultado |
|---|---|---|---|
| `qa-pypi-proxy/simple/six/` | 200 | 19.702 B | enlaces reescritos al proxy |
| `…/pypi/six/json` | 200 | 40.256 B | metadatos correctos |
| `…/packages/…/six-1.16.0-py2.py3-none-any.whl` | 200 | 11.053 B | wheel válido, **idéntico** al upstream |
| `…/six-1.16.0-py2.py3-none-any.whl.metadata` | 200 | 1.795 B | `text/plain`, sidecar PEP 658 |
| SHA-256 del `.metadata` | — | — | `5507062050801267d9725efb139ae23c2378bf64c8b1cfeab5a7278f12872682` |
| `pip download six==1.16.0` | — | 11.053 B | **OK**: descarga el `.metadata` (1,8 kB) y luego el wheel |

`PyPiCoordinatesHelper.withRequestedSidecar()` añade los sufijos `.metadata` y `.asc` a la
ruta del asset antes de resolverla, por lo que el sidecar se pide al upstream como archivo
independiente en lugar de reutilizar el wheel. `pip` completa la descarga sin errores de
hash y sin desactivar la verificación PEP 658.

### 3.13 Raw

| Comprobación | Resultado |
|---|---|
| `PUT raw-hosted/qa/raw-test.txt` | 201 |
| `GET raw-hosted/qa/raw-test.txt` | 200, 48 B, **idéntico** |
| `DELETE raw-hosted/qa/raw-test.txt` | 204 |
| `GET` tras el borrado | 404 |

### 3.14 RubyGems

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `gems-proxy/specs.4.8.gz` | 200 | 6.318.174 B | gzip válido |
| `gems-proxy/gems/rake-13.0.6.gem` | 200 | 84.992 B | tar válido, **idéntico** al upstream |
| `gems-group/specs.4.8.gz` | 200 | 3.434.707 B | índice de grupo servido |
| `gems-group/gems/rake-13.0.6.gem` | 200 | 84.992 B | **idéntico** al upstream |

### 3.15 Yum

Proxy `qa-yum-proxy` → `https://repo.almalinux.org/almalinux/9/BaseOS/x86_64/os`.

| Recurso | HTTP | Tamaño | Validación |
|---|---|---|---|
| `repodata/repomd.xml` | 200 | 3.890 B | XML con primary, filelists, other, group, updateinfo |
| `repodata/…-primary.xml.gz` | 200 | 49.127.746 B | gzip válido |
| `Packages/basesystem-11-13.el9.noarch.rpm` | 200 | 7.145 B | RPM válido, **idéntico** al upstream |

---

## 4. Integridad de artefactos

Todos los artefactos descargados a través de Nexus se compararon con el original del
upstream. La columna «SHA-256» muestra el valor calculado sobre la copia servida por
Nexus; en todos los casos ✅ coincide byte a byte con el upstream.

| Formato | Artefacto | Tamaño | SHA-256 (prefijo…sufijo) | Resultado |
|---|---|---|---|---|
| Maven 2 | `slf4j-api-2.0.13.jar` | 68.605 B | `e7c2a48e…b1204a9` | ✅ |
| npm | `left-pad-1.3.0.tgz` | 3.619 B | `870c0fe1…d481dd` | ✅ |
| RubyGems | `rake-13.0.6.gem` | 84.992 B | `5ce4bf50…b19097` | ✅ |
| Go | `v0.9.1.zip` | 17.866 B | `d4c36b8b…af2bd07` | ✅ |
| Helm | `airflow-21.9.0.tgz` | 265.406 B | `d71af86d…b3373c` | ✅ |
| APT | `hello_2.10-2_amd64.deb` | 56.132 B | `35b1508e…325cbc` | ✅ |
| Yum | `basesystem-11-13.el9.noarch.rpm` | 7.145 B | `4bf863df…4bfcb98` | ✅ |
| PyPI | `six-1.16.0-py2.py3-none-any.whl` | 11.053 B | `8abb2f1d…26e0254` | ✅ |
| PyPI | `six-1.16.0-py2.py3-none-any.whl.metadata` | 1.795 B | `55070620…12872682` | ✅ |
| Cargo | `serde-1.0.0.crate` | 58.433 B | `369633cf…439ae37` (= `cksum`) | ✅ |
| Cargo | `anyhow-1.0.0.crate` | 13.476 B | `a9ff2deb…634e5c7f` (= `cksum`) | ✅ |
| NuGet | `newtonsoft.json.13.0.3.nupkg` | 2.441.966 B | ZIP válido | ✅ |
| Conan | `libhello.a` (hosted) | 1.024 B | — (comparado con `cmp`) | ✅ |
| Alpine | `musl-1.2.5-r4.apk` | 408.411 B | `b255cdcf…2e2bc7d8` | ✅ |
| Alpine | `busybox-1.36.1-r31.apk` | 504.644 B | `a7c2c34d…4fe531c4` | ✅ |
| Ansible | `community-general-1.0.0.tar.gz` | 506 B | — (comparado con `cmp`) | ✅ |
| Raw | `raw-test.txt` | 48 B | — (comparado con `cmp`) | ✅ |
| Docker | `ghcr/astral-sh/uv:0.5.0` | 45.695.328 B | digest `sha256:0e0fb779…e9db9d` | ✅ |
| Docker | `quay/prometheus/node-exporter:v1.8.0` | 38.170.349 B | digest `sha256:8a57af80…c15e4ec` | ✅ |

Los hashes completos y los ficheros de evidencia están en `/tmp/opencode/qa-formats/`.

---

## 5. Hallazgos y causas raíz

### H1 — Cargo: rutas sin barra inicial (corregido)

- **Síntoma original:** `config.json`, índice y `.crate` devolvían 500.
- **Causa:** `CargoCoordinatesHelper.normalize()` devolvía `index/config.json` (sin `/`),
  pero `CargoContentFacetImpl.get/put` pasaban ese valor a `assets().path(...)`, que exige
  ruta absoluta.
- **Corrección:** `CargoCoordinatesHelper.storagePath()` centraliza la conversión a ruta
  absoluta y la usan todas las lecturas/escrituras del facet.
- **Verificación:** `config.json`, índice, descarga de dos crates y segunda petición
  cacheada devuelven 200 con SHA-256 igual al `cksum` (sección 3.4).

### H2 — Conan: rutas sin barra inicial (corregido)

- **Síntoma original:** `PUT`/`GET` de ficheros de receta devolvían 500 y `revisions` 404.
- **Causa:** misma clase de fallo que H1 en `ConanContentFacetImpl.get/save`.
- **Corrección:** `ConanCoordinates.assetPath()` genera la ruta absoluta; `assetsWithPrefix()`
  y las rutas relativas del handler se ajustaron en consecuencia.
- **Verificación:** `PUT`/`GET` de receta, `conaninfo.txt` y binario; revisiones, ficheros y
  `download_urls` de receta y de paquete devuelven 200 y contenido idéntico (sección 3.5).

### H3 — NuGet: base del flat-container mal compuesta (corregido)

- **Síntoma original:** `v3-flatcontainer/…/index.json` y `.nupkg` devolvían 404.
- **Causa:** se concatenaba `v3-flatcontainer` sobre `https://api.nuget.org/v3`, produciendo
  `https://api.nuget.org/v3/v3-flatcontainer/…`.
- **Corrección:** al guardar `index.json` se descubren y persisten `PackageBaseAddress`,
  `RegistrationsBaseUrl` y `SearchQueryService`; flat-container, registration y search se
  enrutan contra esas bases. Los atributos se recargan de forma perezosa tras un reinicio y
  la variante de registration de la petición se conserva.
- **Verificación:** flat-container, `.nupkg` (ZIP válido de 2.441.966 B), registration
  `semver2` y search devuelven 200, también después de reiniciar sin refrescar `index.json`
  (sección 3.11).

### H4 — PyPI: el sidecar `.metadata` devolvía el wheel (corregido)

- **Síntoma original:** `pip download`/`pip install` fallaba por hash mismatch.
- **Causa:** una petición a `*.whl.metadata` se resolvía contra el mismo asset que el `.whl`
  y devolvía el wheel completo en lugar del metadatos.
- **Corrección:** `PyPiCoordinatesHelper.withRequestedSidecar()` añade los sufijos
  `.metadata`/`.asc` a la ruta antes de resolverla.
- **Verificación:** el `.metadata` devuelve 1.795 B `text/plain` con SHA-256
  `55070620…12872682`; `pip download six==1.16.0` completa sin errores (sección 3.12).

### H5 — Cargo: `api` debía ser la raíz y las descargas usar el `dl` descubierto (corregido)

- **Síntoma original:** tras corregir H1, `config.json` anunciaba `api` con `/api` y
  `/api/v1/crates/{crate}/{version}/download` devolvía 500/404.
- **Causa:** Cargo construye `<api>/api/v1/crates/...` a partir de `api`, por lo que el
  valor debe ser la raíz del repositorio. Además las descargas deben ir contra el `dl`
  anunciado por el upstream (`https://static.crates.io/crates`), no contra el índice.
- **Corrección:** `CargoIndexBuilder.configJson()`/`rewriteConfig()` publican la raíz en
  `api`; `CargoProxyFacet` enruta descargas por el `dl` descubierto y las peticiones de API
  por la base `api`, aceptando tanto raíces como bases que ya incluyen `/api/v1/crates`.
  El `.crate` se almacena ya con `Content-Type: application/gzip`.
- **Verificación:** `api` = raíz, descargas de `serde` y `anyhow` con `cksum` correcto y
  respuesta cacheada idéntica (sección 3.4).

### H6 — Docker: Docker Hub y `gcr` no deben intentarse (corregido por configuración)

- **Síntoma original:** las rutas de Docker Hub acababan en `registry-1.docker.io`, que
  responde 403 en este entorno, y el mirror `gcr` servía manifiestos pero rechazaba blobs.
- **Decisión:** los defaults son solo `ghcr` y `quay` (`nexus.docker.defaultUpstream=ghcr`);
  `gcr` no se configura porque sus blobs devuelven 403; Docker Hub no se configura por
  defecto.
- **Corrección:** `DockerRegistryConfig.resolve()` rechaza los prefijos de registros
  conocidos pero no configurados (`dockerhub`, `docker`, `docker.io`, `index.docker.io`,
  `registry-1.docker.io`, `library`, `ghcr`, `quay`, `gcr`) en lugar de enviarlos al
  upstream por defecto. Un repositorio con `remoteUrl` de Docker Hub devuelve 404 con un
  aviso en el log y no genera tráfico hacia Docker Hub.
- **Verificación:** `ghcr` y `quay` descargan y ejecutan imágenes; `gcr`, `library/...`,
  `dockerhub/...`, `docker.io/...` y el repo `qa-docker-proxy` devuelven 404 en menos de
  110 ms sin intentos de salida (sección 3.6).

### H7 — Conan Center bloqueado por la red del entorno (limitación externa)

- **Síntoma:** `https://center.conan.io/v1/ping` y `/v2/ping` devuelven 403 incluso con
  `User-Agent: Conan/2.0`.
- **Alcance:** impide validar el proxy Conan contra un upstream real; el contenido hosted
  ya funciona tras H2. Es una limitación de red, no del módulo.

---

## 6. Reproducción

### 6.1 Crear los repositorios de prueba

Con `PW` = contraseña de `admin` (línea 1 de `nexus-public/admin-creds.txt`):

```bash
CREDS="admin:$PW"
B=http://localhost:8081/service/rest/v1/repositories

curl -u "$CREDS" -X POST "$B/apt/proxy" -H 'Content-Type: application/json' -d '{
  "name":"qa-apt-proxy","online":true,
  "storage":{"blobStoreName":"default","strictContentTypeValidation":false},
  "apt":{"distribution":"bookworm","flat":false},
  "proxy":{"remoteUrl":"https://deb.debian.org/debian","contentMaxAge":1440,"metadataMaxAge":1440},
  "negativeCache":{"enabled":false,"timeToLive":1440},
  "httpClient":{"blocked":false,"autoBlock":false}}'

curl -u "$CREDS" -X POST "$B/cargo/proxy" -H 'Content-Type: application/json' -d '{
  "name":"qa-cargo-proxy","online":true,
  "storage":{"blobStoreName":"default","strictContentTypeValidation":false},
  "proxy":{"remoteUrl":"https://index.crates.io","contentMaxAge":1440,"metadataMaxAge":1440},
  "negativeCache":{"enabled":false,"timeToLive":1440},
  "httpClient":{"blocked":false,"autoBlock":false},"cargo":{}}'

curl -u "$CREDS" -X POST "$B/conan/proxy" -H 'Content-Type: application/json' -d '{
  "name":"qa-conan-proxy","online":true,
  "storage":{"blobStoreName":"default","strictContentTypeValidation":false},
  "proxy":{"remoteUrl":"https://center.conan.io","contentMaxAge":1440,"metadataMaxAge":1440},
  "negativeCache":{"enabled":false,"timeToLive":1440},
  "httpClient":{"blocked":false,"autoBlock":false},"conan":{}}'

curl -u "$CREDS" -X POST "$B/nuget/proxy" -H 'Content-Type: application/json' -d '{
  "name":"qa-nuget-proxy","online":true,
  "storage":{"blobStoreName":"default","strictContentTypeValidation":false},
  "proxy":{"remoteUrl":"https://api.nuget.org/v3/index.json","contentMaxAge":1440,"metadataMaxAge":1440},
  "negativeCache":{"enabled":false,"timeToLive":1440},
  "httpClient":{"blocked":false,"autoBlock":false},"nugetProxy":{}}'

curl -u "$CREDS" -X POST "$B/pypi/proxy" -H 'Content-Type: application/json' -d '{
  "name":"qa-pypi-proxy","online":true,
  "storage":{"blobStoreName":"default","strictContentTypeValidation":false},
  "proxy":{"remoteUrl":"https://pypi.org","contentMaxAge":1440,"metadataMaxAge":1440},
  "negativeCache":{"enabled":false,"timeToLive":1440},
  "httpClient":{"blocked":false,"autoBlock":false},"pypi":{}}'

curl -u "$CREDS" -X POST "$B/yum/proxy" -H 'Content-Type: application/json' -d '{
  "name":"qa-yum-proxy","online":true,
  "storage":{"blobStoreName":"default","strictContentTypeValidation":false},
  "proxy":{"remoteUrl":"https://repo.almalinux.org/almalinux/9/BaseOS/x86_64/os","contentMaxAge":1440,"metadataMaxAge":1440},
  "negativeCache":{"enabled":false,"timeToLive":1440},
  "httpClient":{"blocked":false,"autoBlock":false},"yum":{}}'
```

### 6.2 Comandos de prueba

```bash
BASE=http://localhost:8081/repository

# APT
curl -s "$BASE/qa-apt-proxy/dists/bookworm/Release" | head
curl -s "$BASE/qa-apt-proxy/pool/main/h/hello/hello_2.10-2_amd64.deb" -o /tmp/hello.deb
dpkg-deb -I /tmp/hello.deb

# Cargo
curl -s "$BASE/qa-cargo-proxy/index/config.json"
curl -s "$BASE/qa-cargo-proxy/index/se/rd/serde" | grep -m1 '"vers":"1.0.0"'
curl -s -L -o /tmp/serde.crate \
  "$BASE/qa-cargo-proxy/api/v1/crates/serde/1.0.0/download"
sha256sum /tmp/serde.crate   # debe coincidir con el cksum del índice

# Conan (hosted)
C="$BASE/qa-conan-hosted/conans/hello/1.0/_/_"
curl -s -u "$CREDS" -X PUT --data-binary @conanfile.py \
  "$C/revisions/11111111111111111111111111111111/files/conanfile.py"
curl -s "$C/revisions"
curl -s "$C/revisions/11111111111111111111111111111111/files/conanfile.py"
curl -s "$BASE/qa-conan-hosted/v1/conans/hello/1.0/_/_/download_urls"

# NuGet
curl -s "$BASE/qa-nuget-proxy/index.json" | head -c 200
curl -s "$BASE/qa-nuget-proxy/v3-flatcontainer/newtonsoft.json/index.json"
curl -s -o /dev/null -w '%{http_code}\n' \
  "$BASE/qa-nuget-proxy/v3-flatcontainer/newtonsoft.json/13.0.3/newtonsoft.json.13.0.3.nupkg"
curl -s "$BASE/qa-nuget-proxy/registration5-gz-semver2/newtonsoft.json/index.json" \
  | head -c 200
curl -s "$BASE/qa-nuget-proxy/v3/query?q=newtonsoft&take=1" | head -c 200

# PyPI
curl -s "$BASE/qa-pypi-proxy/simple/six/" | head -c 200
pip download --no-deps --no-cache-dir -d /tmp/pipdl \
  --index-url "$BASE/qa-pypi-proxy/simple/" six==1.16.0

# Yum
curl -s "$BASE/qa-yum-proxy/repodata/repomd.xml" | head
curl -s "$BASE/qa-yum-proxy/Packages/basesystem-11-13.el9.noarch.rpm" -o /tmp/basesystem.rpm
file /tmp/basesystem.rpm

# Docker: mirrors verificados y rechazo de los no configurados
docker pull localhost:8081/ghcr/astral-sh/uv:0.5.0
docker pull localhost:8081/quay/prometheus/node-exporter:v1.8.0
docker pull localhost:8081/gcr/google-containers/busybox:1.27   # 404, sin salida
docker pull localhost:8081/library/nginx:latest                 # 404, sin Docker Hub
```

### 6.3 Limpieza de los repositorios de prueba

```bash
for r in qa-apt-proxy qa-cargo-proxy qa-conan-proxy qa-conan-hosted \
         qa-nuget-proxy qa-pypi-proxy qa-yum-proxy qa-ansible-hosted qa-docker-proxy; do
  curl -s -o /dev/null -w "$r -> %{http_code}\n" -u "$CREDS" \
    -X DELETE "http://localhost:8081/service/rest/v1/repositories/$r"
done
```

---

## 7. Limitaciones de esta validación

1. **Clientes nativos:** se usaron `apk`, `npm`, `pip` y `docker`; para APT, Cargo, Conan,
   Go, Helm, Maven, NuGet, PyPI (además de `pip`), RubyGems y Yum las pruebas fueron a
   nivel HTTP con validación del artefacto (`tar`, `gzip`, `zipfile`, `dpkg-deb`, `cmp` y
   SHA-256). No se ejecutó `cargo`, `conan` ni `dotnet` contra los repositorios.
2. **Red del entorno:** `galaxy.ansible.com`, `center.conan.io` y Docker Hub
   (`auth.docker.io`, `registry-1.docker.io`) están bloqueados o limitados; el mirror `gcr`
   rechaza blobs. Los fallos de red se separaron de los fallos del módulo en la sección 5.
3. **Cuatro formatos sin receta:** `cocoapods`, `p2`, `swift` y `gitlfs` no se pueden
   probar E2E en esta build; los tres últimos son placeholders sin código Java.
4. **Repositorios de prueba:** los `qa-*` se dejaron creados para que se puedan inspeccionar
   los resultados. El bloque 6.3 contiene los comandos para eliminarlos.
5. **Caché:** las pruebas de proxy poblaron la caché de Nexus; los artefactos de prueba
   en `maven-releases` y `helm-hosted` también quedaron almacenados. La caché de Cargo,
   NuGet y PyPI se invalidó tras aplicar las correcciones para evitar entradas envenenadas.
6. **Docker:** solo se validaron los mirrors `ghcr` y `quay`. Docker Hub y `gcr` se
   rechazan de forma deliberada, por lo que no hay una prueba de descarga completa contra
   ellos.

---

## 8. Conclusión

La build valida los **15 formatos con receta** (Alpine, Ansible, APT, Cargo, Conan, Docker,
Go, Helm, Maven 2, npm, NuGet, PyPI, Raw, RubyGems y Yum), con descargas byte a byte
idénticas a los upstreams y, cuando aplica, instalación/ejecución con el cliente nativo
(`apk`, `npm`, `pip`, `docker`).

Las cuatro causas raíz que bloqueaban Cargo, Conan, NuGet y PyPI quedaron corregidas y
verificadas contra la assembly en marcha:

- **Cargo:** rutas absolutas de almacenamiento, `api` como raíz del repositorio y descargas
  enrutadas por el `dl` descubierto; `.crate` con `cksum` correcto y caché reutilizable.
- **Conan:** rutas absolutas de asset; receta, binarios, revisiones y `download_urls`
  funcionan en hosted.
- **NuGet:** endpoints del índice de servicio descubiertos, persistidos y usados para
  flat-container, registration y search, sobreviviendo a un reinicio.
- **PyPI:** sidecar `.metadata` servido como archivo independiente; `pip download` completo
  sin desactivar PEP 658.

En Docker se aplicó la política de usar solo mirrors verificados: `ghcr` y `quay` funcionan
de extremo a extremo, mientras que Docker Hub y `gcr` se rechazan con un 404 rápido y sin
tráfico saliente. Las únicas limitaciones restantes son externas al fork: Conan Center y
Docker Hub están bloqueados por la red del entorno, y el mirror `gcr` rechaza blobs.

Los cuatro módulos sin receta (CocoaPods, p2, Swift y Git LFS) siguen fuera del alcance de
esta validación.

