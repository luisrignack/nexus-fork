# Análisis del Proyecto — Fork de Nexus Repository

> Fecha de generación: 2026-10-24  
> Directorio raíz: `/home/ladmin/Downloads/nexus-fork`

---

## 1. Estructura general del workspace

```text
/home/ladmin/Downloads/nexus-fork/
├── .git/                        # Repo raíz del workspace (documentación del proyecto)
├── .gitignore                   # Ignora .ssh/, toolchain/, tokens, *.bak, credenciales
├── ANALISIS_PROYECTO.md        # Este documento de resumen y estado actual
├── ESTADO_PROYECTO.md          # Bitácora detallada e historial del proyecto
├── ESTADO_PROYECTO.md.bak      # Copia de respaldo de la bitácora (ignorada por git)
├── docs/                       # Documentación técnica y de diseño
│   ├── DISENO_PLUGIN_NPM.md    # Diseño del plugin npm (F1-F4)
│   ├── DISENO_PLUGIN_YUM.md    # Diseño del plugin Yum/RPM
│   ├── DOCKER.md               # Proxy Docker/OCI, configuración y caché
│   ├── FUNCIONALIDADES_PRO.md  # Catálogo priorizado de funciones Pro/Pro+ + informe §8
│   ├── LDAP.md                 # Realm, configuración y pantalla de administración LDAP
│   ├── POSTGRESQL.md           # Guía de configuración y uso con PostgreSQL
│   └── VALIDACION_HELM_GO_RUBYGEMS.md  # Validación E2E de Helm, Go y RubyGems
├── toolchain/                  # Herramientas locales (sin requerir sudo) — ignorado
│   ├── jdk-21.0.13+9/          # Eclipse Temurin JDK 21 (JAVA_HOME)
│   ├── apache-maven-3.9.16/    # Apache Maven 3.9.16
│   └── bin/                    # Shims de corepack (yarn, pnpm, etc.)
├── nexus-public/               # Submódulo: fork de sonatype/nexus-public 3.89.0-09 (21 commits propios)
├── .ssh/                       # Claves SSH locales para GitHub (ignorado por git)
├── CLAVE_PUBLICA_GITHUB.txt    # Clave pública ed25519 asociada a GitHub
└── button-color-sample.html    # Muestra HTML auxiliar
```

---

## 2. Propósito y alcance del proyecto

El objetivo es mantener un **fork de Sonatype Nexus Repository Core (`3.89.0-09`, licencia EPL-1.0)** que incorpore las capacidades de las ediciones completas (Community Edition y funciones tipo Pro/Pro+ reimplementadas de forma limpia desde especificaciones públicas), superando las limitaciones del repositorio abierto original (que solo incluye los formatos Maven, Raw y APT sobre H2 embebido).

---

## 3. Estado de los formatos de repositorio

Ubicación en el árbol: `nexus-public/public/common/components/formats/`

| Formato | Módulo | Tipos soportados | Estado |
|---|---|---|---|
| **Maven** | `nexus-repository-maven` | `hosted`, `proxy`, `group` | Original upstream OSS |
| **APT** | `nexus-repository-apt` | `hosted`, `proxy` | Original upstream OSS |
| **Raw** | `nexus-repository-raw` | `hosted`, `proxy`, `group` | Original upstream OSS |
| **npm** | `nexus-repository-npm` | `hosted`, `proxy`, `group` | **Completo y verificado** (publish, packuments dinámicos, dist-tags, unpublish, paquetes con `@scope`, búsqueda `/-/v1/search`, `npm audit`, tarea de rebuild de metadata) |
| **Docker / OCI** | `nexus-repository-docker` | `proxy` (pull-through) | **Completo y verificado** (`/v2/*`, formato `docker` y receta `docker-proxy` visibles en UI y REST API, caché en disco con TTL/sidecars, soporte de mirrors y credenciales) |
| **PyPI** | `nexus-repository-pypi` | `hosted`, `proxy`, `group` | **Completo y verificado** (upload estilo twine, índices PEP 503 `/simple/`, reescritura en proxy y fusión en group) |
| **Yum / RPM** | `nexus-repository-yum` | `hosted`, `proxy` | **Completo y verificado** (parser de cabeceras RPM propio, generación de `repomd.xml`, `primary.xml.gz`, `filelists.xml.gz`, `other.xml.gz`) |
| **Helm** | `nexus-repository-helm` | `hosted`, `proxy`, `group` | **Completo y validado E2E** (chart propio + índice Bitnami vía proxy, merge de grupos; `docs/VALIDACION_HELM_GO_RUBYGEMS.md`) |
| **Go** | `nexus-repository-go` | `hosted`, `proxy`, `group` | **Completo y validado E2E** (`.mod`/`.info`/`.zip` contra `proxy.golang.org`) |
| **RubyGems** | `nexus-repository-rubygems` | `hosted`, `proxy`, `group` | **Completo y validado E2E** (`.gem` propios y descarga de `rubygems.org`) |
| **NuGet** | *(pendiente)* | — | Pendiente de implementación (ver `docs/FUNCIONALIDADES_PRO.md` §8.5) |

---

## 4. Base de datos y funciones tipo Pro incorporadas

- **PostgreSQL externo:**
  - Driver `org.postgresql:postgresql` incluido como dependencia `runtime` en la assembly (`BOOT-INF/lib`).
  - Clase `DataStoreConfigurationEnvironmentSource` (`nexus-datastore-h2`) para configurar `NEXUS_DATASTORE_NEXUS_JDBCURL`, `USERNAME`, `PASSWORD`, `SCHEMA` y `ADVANCED` mediante variables de entorno o propiedades de sistema.
  - Verificado con PostgreSQL 15 externo.
- **User Tokens (función Pro):**
  - Backend completo implementado en `nexus-security` y `nexus-coreui-plugin` (`UserTokenService`, `UserTokenRealm`, `UserTokenEnforcementFilter`, `UserTokenCapability`, recursos REST V1/Beta en `/service/rest/v1/security/user-tokens` y `/user-tokens/current`).
  - Eliminado el bloqueo por licencia/edición Pro en las rutas de administración (`/admin/security/usertoken`) y perfil de usuario.
- **SSO OIDC/OAuth2 (función Pro, commit `51add5a`):** realm `OAuth2Realm`, cliente OIDC con
  validación JWKS/`iss`/`aud`/`exp`/`nonce`, flujo `/oidc/login` + `/oidc/callback` con
  `state`/`nonce` de un solo uso, capability `oidc`, UI des-gated y provisioning JIT con mapeo de
  grupos. Ver `docs/FUNCIONALIDADES_PRO.md` §3.1.
- **LDAP (función Pro, commits `fa46a5e` + `2df79cf`):** realm `LdapRealm`, configuración y
  directorio sobre capability `ldap`, REST `/service/rest/v1/security/ldap`, componente Ext.Direct
  `ldap_LdapServer` y pantalla React de 5 pasos con 4 plantillas (AD, Posix Static/Dynamic,
  Generic). Privilegios `nx-ldap-*`. Ver `docs/LDAP.md`.
- **Operación (commits `5552712`, `8913d17`, `fe3e0e4`):** GC de la caché Docker
  (`repository.docker-gc`), exportación de SBOM CycloneDX 1.5 / SPDX 2.3 (`nexus-sbom`) y backup de
  blobs + datastore con manifiesto verificado (`nexus-backup`).
- **Correcciones de interfaz (CoreUI):**
  - Ajuste del formulario `npm_proxy` y `docker_proxy` para no requerir facets propietarios ausentes.
  - Ocultación de la ruta de administración LDAP cuando el bundle backend no está cargado para evitar fallos en el navegador (retirada en `2df79cf`, ya con backend propio).

---

## 5. Estado del repositorio Git (`nexus-public`)

- **Remotos configurados:**
  - `origin`: `git@github.com:luisrignack/nexus-public.git` (**público**)
  - `upstream`: `https://github.com/sonatype/nexus-public.git`
- **Rama actual:** `main`, sincronizada con `origin/main` y con el working tree limpio. **21 commits**
  propios desde el punto de partida upstream (`0a8a425`, 3.89.0-09). Últimos commits:
  1. `2df79cf` — `feat(ldap): wire the LDAP admin screen into the React UI`
  2. `fa46a5e` — `feat(ldap): add LDAP authentication realm, configuration and admin REST API`
  3. `e4055d9` — `fix(content,helm): asset re-publish and helm index merge`
  4. `10aa594` — `chore(security): stop tracking admin-creds.txt and ignore it`
  5. `51add5a` — `feat(oidc): add OIDC/OAuth2 SSO with realm, login flow and configuration UI`
  6. `fe3e0e4` — `feat(backup): add datastore and blob-store backup task with verified manifest`
  7. `8913d17` — `feat(sbom): add CycloneDX 1.5 and SPDX 2.3 inventory export REST endpoint`
  8. `5552712` — `feat(docker): add cache garbage collection task (repository.docker-gc)`
  9. `fbb6031` — `fix(search-sql): skip reindex checks for repositories without ContentFacet`
  10. `454338a` — `feat(usertoken): add the user token backend and its privileges`
- **Autenticación de push:** SSH con `core.sshCommand=ssh -F /home/ladmin/Downloads/nexus-fork/.ssh/config`
  (identidad `luisrignack`). El repo raíz del workspace usa `GIT_SSH_COMMAND` con el mismo fichero.
- **Repo raíz del workspace:** `/home/ladmin/Downloads/nexus-fork` es un repositorio git propio con
  la documentación; `nexus-public` está registrado como submódulo.

---

## 6. Comandos rápidos de compilación y ejecución

```bash
cd /home/ladmin/Downloads/nexus-fork/nexus-public
export JAVA_HOME=/home/ladmin/Downloads/nexus-fork/toolchain/jdk-21.0.13+9
export PATH=/home/ladmin/Downloads/nexus-fork/toolchain/bin:$JAVA_HOME/bin:/home/ladmin/Downloads/nexus-fork/toolchain/apache-maven-3.9.16/bin:$PATH
export MAVEN_OPTS="-Xmx3g -Djava.net.preferIPv4Stack=true"

# Compilar todo el proyecto y generar la assembly
mvn -o -B install -Ppublic -DskipTests -Dmaven.javadoc.skip=true

# Ejecutar la instancia compilada (http://localhost:8081)
cd public/selfhosted/assemblies/nexus-repository-core/target/assembly
./bin/nexus run
```

---

## 7. Próximos pasos sugeridos

1. ~~**Cerrar cambio en curso:** Ejecutar los tests de `nexus-search-sql` y commitear el arreglo de `SqlSearchUpdateService`, actualizando `ESTADO_PROYECTO.md` con los últimos avances (`usertoken`, fix UI LDAP y fix SQL search).~~ ✅ **Completado (`cfe3701`)**
2. **Funciones P0 (según `docs/FUNCIONALIDADES_PRO.md`):**
   - ~~Tareas de mantenimiento y Garbage Collection (GC) de la caché Docker~~ ✅ **Completado (`5552712`).**
   - ~~Exportación de SBOM (CycloneDX / SPDX)~~ ✅ **Completado (`8913d17`).**
   - ~~Backup y restore consistente (base de datos + blob stores)~~ ✅ **Completado (`fe3e0e4`);** falta la tarea de *restore*.
   - ~~Autenticación SSO mediante OIDC~~ ✅ **Completado (2026-10-24).** Sigue **SAML 2.0** como único SSO pendiente (ver `docs/FUNCIONALIDADES_PRO.md` §8.2).
   - ~~Pantalla y API de administración LDAP~~ ✅ **Completado (`fa46a5e` + `2df79cf`).** Ver `docs/LDAP.md`.
3. **Formatos pendientes:**
   - ~~**Validar end-to-end y documentar Helm, Go y RubyGems**~~ ✅ **Completado (2026-10-24):** ver `docs/VALIDACION_HELM_GO_RUBYGEMS.md` — E2E en verde tras corregir dos defectos de regresión (re-publicación de assets con atributos escalares y merge de índices de grupo de Helm), commit `e4055d9` pushed a `origin/main`.
   - Diseño e implementación del formato **NuGet** (siguiente formato por demanda; resto en `docs/FUNCIONALIDADES_PRO.md` §8.5).
4. ~~**Sincronización con GitHub:** Subir los commits acumulados en `main` hacia `origin`~~ ✅ **Completado (2026-10-24): `main` está al día con `origin/main` (21 commits).**
5. **Repo raíz del workspace:** publicar `ANALISIS_PROYECTO.md`, `ESTADO_PROYECTO.md` y `docs/` en un repositorio propio con `nexus-public` como submódulo (requiere crear el repositorio en GitHub con un PAT).
6. **Seguridad pendiente:** el commit `be30239` cambió la contraseña de admin y `admin-creds.txt` dejó de seguirse en `10aa594`; **rotar la contraseña filtrada en el historial de Git** y verificar que no haya secretos en los primeros 18 commits. Los docs ya no contienen credenciales (sustituidas por `$NEXUS_PASSWORD`).
7. **Siguientes pasos de producto (orden recomendado):** SAML → tag management → NuGet → Azure/GCS → soft-delete → SCIM → malware/auditoría. Detalle y esfuerzos en `docs/FUNCIONALIDADES_PRO.md` §8.7.
