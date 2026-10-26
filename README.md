# nexus-fork

Fork de **Sonatype Nexus Repository Core 3.89.0-09** (EPL-1.0) que añade los formatos y
funciones que el repositorio público original no incluye: npm, Yum, PyPI, Helm, Go, RubyGems,
Docker/OCI, Ansible, Alpine, NuGet, Cargo y Conan, además de PostgreSQL externo y varias
capacidades tipo Pro reimplementadas desde especificaciones públicas.

> **Estado (2026-10-26):** los 15 formatos con receta están implementados, desplegados en la
> assembly y validados de extremo a extremo. Los 4 módulos sin receta (CocoaPods, p2, Swift y
> Git LFS) siguen pendientes. Resultados completos en [`PRUEBAS_FORMATOS.md`](PRUEBAS_FORMATOS.md).

## Contenido del repositorio

Este repositorio contiene la **documentación y el entorno del proyecto**; el código vive en el
submódulo [`nexus-public`](https://github.com/luisrignack/nexus-public), que es el fork real de
`sonatype/nexus-public`.

```text
nexus-fork/
├── README.md                  # este documento
├── PRUEBAS_FORMATOS.md        # batería E2E de los 15 formatos con receta
├── ESTADO_PROYECTO.md         # bitácora detallada y estado por sesión
├── ANALISIS_PROYECTO.md       # análisis del fork y de las ediciones de Nexus
├── docs/                      # documentación técnica por formato y función
│   ├── FORMATOS.md
│   ├── DOCKER.md
│   ├── POSTGRESQL.md
│   ├── LDAP.md
│   ├── FUNCIONALIDADES_PRO.md
│   └── ...
└── nexus-public/              # submódulo con el código del fork
```

## Requisitos

- **JDK 21** (probado con Eclipse Temurin `21.0.13+9`).
- **Apache Maven 3.9+** (el repositorio no incluye `mvnw`).
- **Node.js 20+ y Yarn 4** para recompilar la UI.
- **Git** con acceso al submódulo.

La compilación completa necesita descargar dependencias de Maven Central y paquetes de Yarn.

## Clonar

```bash
git clone --recurse-submodules git@github.com:luisrignack/nexus-fork.git
cd nexus-fork
```

Si ya lo clonaste sin submódulos:

```bash
git submodule update --init --recursive
```

## Compilar

```bash
export JAVA_HOME=/ruta/al/jdk-21
export PATH="$JAVA_HOME/bin:$PATH"

cd nexus-public
mvn -B clean install -Ppublic -DskipTests -Dmaven.javadoc.skip=true
```

El artefacto queda en
`nexus-public/public/selfhosted/assemblies/nexus-repository-core/target/assembly/`.

### Compilar solo un formato

Para iterar sobre un módulo sin recompilar todo el reactor:

```bash
cd nexus-public

# tests del módulo
mvn -o -q -pl public/common/components/formats/nexus-repository-<formato> test

# instalar el jar corregido
mvn -o -q -pl public/common/components/formats/nexus-repository-<formato> -DskipTests install

# regenerar la assembly (borrar antes el uberjar evita reutilizar el artefacto viejo)
rm -f public/selfhosted/assemblies/nexus-repository-core/target/nexus-repository-core-*.jar{,.original}
mvn -o -q -pl public/selfhosted/assemblies/nexus-repository-core -DskipTests package
```

> `nexus-repository-cocoapods` rompe los builds que usan `-am` porque su jar está incompleto.
> Compila los módulos de formato por separado y empaqueta la assembly sola, como en el ejemplo.

## Ejecutar

```bash
cd nexus-public/public/selfhosted/assemblies/nexus-repository-core/target/assembly
./bin/nexus run
```

- UI y API: <http://localhost:8081>
- Estado: `GET /service/rest/v1/status`
- Contraseña inicial de `admin`: `target/sonatype-work/nexus3/admin.password`

`bin/nexus stop` no funciona en esta assembly; detén el proceso con `kill <pid>`.

## Tests

```bash
cd nexus-public

mvn -o -q -pl public/common/components/formats/nexus-repository-npm test
mvn -o -q -pl public/common/components/formats/nexus-repository-nuget test
mvn -o -q -pl public/common/components/formats/nexus-repository-docker test
```

Cobertura actual de los módulos de formato tocados en la última sesión:

| Módulo | Tests |
|---|---:|
| `nexus-repository-cargo` | 26 |
| `nexus-repository-conan` | 47 |
| `nexus-repository-nuget` | 30 |
| `nexus-repository-pypi` | 4 |
| `nexus-repository-docker` | 32 |

## Formatos

| Formato | Recetas | Estado |
|---|---|---|
| `maven2` | hosted, proxy, group | ✅ base OSS |
| `raw` | hosted, proxy, group | ✅ base OSS |
| `apt` | hosted, proxy | ✅ base OSS |
| `npm` | hosted, proxy, group | ✅ fork + E2E |
| `yum` | hosted, proxy | ✅ fork + E2E |
| `pypi` | hosted, proxy, group | ✅ fork + E2E |
| `helm` | hosted, proxy, group | ✅ fork + E2E |
| `go` | hosted, proxy, group | ✅ fork + E2E |
| `rubygems` | hosted, proxy, group | ✅ fork + E2E |
| `docker` | proxy | ✅ fork + E2E |
| `ansible` | hosted, proxy, group | ✅ fork + E2E hosted |
| `alpine` | hosted, proxy, group | ✅ fork + E2E |
| `nuget` | hosted, proxy, group | ✅ fork + E2E |
| `cargo` | hosted, proxy, group | ✅ fork + E2E |
| `conan` | hosted, proxy, group | ✅ fork + E2E hosted |
| `cocoapods` | proxy | ⏳ pendiente |
| `p2` | proxy | ⏳ pendiente |
| `swift` | hosted, proxy | ⏳ pendiente |
| `gitlfs` | hosted | ⏳ pendiente |

Detalle de endpoints, almacenamiento y verificación en [`docs/FORMATOS.md`](docs/FORMATOS.md).

### Docker/OCI

El proxy Docker se monta en `/v2/` y solo usa mirrors verificados:

- `ghcr` y `quay` están habilitados por defecto (`nexus.docker.defaultUpstream=ghcr`).
- Docker Hub **no** se configura por defecto; sus rutas devuelven 404 rápido y no generan
  tráfico hacia `registry-1.docker.io`.
- `gcr` tampoco se usa por defecto porque su mirror sirve manifiestos pero rechaza blobs.
- Los prefijos de registros conocidos no configurados se rechazan con 404 en lugar de enviarse
  al upstream por defecto.

Configuración, caché y limitaciones en [`docs/DOCKER.md`](docs/DOCKER.md).

## Funciones tipo Pro incluidas

- PostgreSQL externo (`docs/POSTGRESQL.md`).
- LDAP: realm, configuración y pantalla de administración (`docs/LDAP.md`).
- SSO OIDC y User Tokens.
- SBOM CycloneDX/SPDX, backup/restore con manifiesto y GC de la caché Docker.
- Catálogo completo y estado en [`docs/FUNCIONALIDADES_PRO.md`](docs/FUNCIONALIDADES_PRO.md).

## Limitaciones conocidas

- **Red del entorno:** Docker Hub, `center.conan.io` y `galaxy.ansible.com` están bloqueados o
  limitados; el mirror `gcr` rechaza blobs. Por eso Docker Hub y `gcr` no se usan por defecto y
  el proxy Conan no se pudo validar contra un upstream real.
- **Docker no soporta push:** es un proxy pull-through de solo lectura.
- **CocoaPods, p2, Swift y Git LFS:** sin receta registrada todavía.
- **Alpine:** solo APK v2; el índice generado no va firmado.
- **HA:** la caché Docker es local a cada nodo.

## Documentación

| Documento | Contenido |
|---|---|
| [`PRUEBAS_FORMATOS.md`](PRUEBAS_FORMATOS.md) | Resultados E2E, integridad de artefactos y hallazgos |
| [`ESTADO_PROYECTO.md`](ESTADO_PROYECTO.md) | Bitácora del proyecto, decisiones y estado por sesión |
| [`ANALISIS_PROYECTO.md`](ANALISIS_PROYECTO.md) | Análisis del fork frente a las ediciones de Nexus |
| [`docs/FORMATOS.md`](docs/FORMATOS.md) | Formatos, endpoints y almacenamiento |
| [`docs/DOCKER.md`](docs/DOCKER.md) | Proxy Docker/OCI |
| [`docs/POSTGRESQL.md`](docs/POSTGRESQL.md) | Configuración con PostgreSQL externo |
| [`docs/LDAP.md`](docs/LDAP.md) | Integración LDAP |
| [`docs/FUNCIONALIDADES_PRO.md`](docs/FUNCIONALIDADES_PRO.md) | Funciones Pro/Pro+ y estado |

## Licencia

El código del submódulo `nexus-public` se distribuye bajo **Eclipse Public License 1.0**
(EPL-1.0), igual que el proyecto original de Sonatype; consulta
[`nexus-public/LICENSE.txt`](https://github.com/luisrignack/nexus-public/blob/main/LICENSE.txt)
y los avisos de atribución del upstream. Las funciones añadidas se implementaron desde
especificaciones públicas, sin portar código propietario de Sonatype.
