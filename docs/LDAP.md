# LDAP — realm, configuración y pantalla de administración

> Estado: **funcional y verificado** en la instancia local (`http://localhost:8081`).
> Commits: `fa46a5e` (backend) y `2df79cf` (UI React).
> Licencia: EPL-1.0. Reimplementación limpia desde el comportamiento observable de la API pública;
> no se ha copiado código de la edición comercial `nexus-ldap-plugin`.

## 1. Qué hay implementado

| Capa | Dónde | Qué hace |
|---|---|---|
| Realm Shiro | `nexus-security` → `org.sonatype.nexus.security.authc.ldap.LdapRealm` | Autentica contra un servidor LDAP/AD; mapea grupos del directorio a roles Nexus. |
| Configuración | `nexus-security` → `LdapConfiguration`, `LdapConfigurationStore`, `LdapDirectory` | Contrato de la configuración del directorio y del conector. |
| Capability | `nexus-coreui-plugin` (módulo Java) → `internal.security.ldap.LdapCapability*` | Persiste la configuración en `CapabilityRegistry` con la clave `ldap`; cifrado de credenciales con `SecretsService` y placeholder en la UI. |
| Almacenamiento/directorio | `LdapConfigurationData`, `LdapConfigurationStoreImpl`, `LdapDirectoryImpl` | Carga/guarda la config, conecta (bind), resuelve usuarios y grupos, paginación y filtros. |
| REST de administración | `internal.security.ldap.rest.LdapServerApiResource` | CRUD de servidores LDAP por nombre, ordenación, pruebas de conexión/mapeo/login. |
| Ext.Direct (UI) | `internal.security.ldap.LdapServerComponent` (`@DirectAction("ldap_LdapServer")`) | Acciones que consume la pantalla React: plantillas, verificaciones y limpieza de caché. |
| Plantillas | `LdapSchemaTemplateXO` | 4 esquemas precargados: **Active Directory**, **Posix Static**, **Posix Dynamic**, **Generic**. |
| Privilegios | `NexusSecurityContributor` | `nx-ldap-all`, `nx-ldap-create`, `nx-ldap-delete`, `nx-ldap-read`, `nx-ldap-update`. |
| UI React | `nexus-coreui-plugin` frontend → `admin.security.ldap` | Pantalla de administración de 5 pasos con asistente de conexión, mapeo de usuario y prueba de login. |

## 2. Por qué hizo falta `2df79cf`

La ruta `/admin/security/ldap` estaba ocultada en `e5dff6b` porque apuntaba al bundle
`nexus-ldap-plugin`, que **no existe en la edición pública**: el menú no aparecía y, al forzar la
URL, la pantalla fallaba. El arreglo en `adminRoutes.js` fue retirar ese `bundle` de
`visibilityRequirements` y dejar la ruta visible con permisos reales.

El backend sí estaba (a partir de `fa46a5e`), pero la pantalla React hacía llamadas que devolvían
error:

| Llamada | Antes | Después |
|---|---|---|
| `GET /service/rest/v1/security/ldap/{name}` | 405 (no existía) | 200 / 404 |
| `PUT .../ldap/{name}` (renombrar) | 405 | 200 con el nuevo nombre (fallback a `xo.getId()` en el cuerpo) |
| `DELETE .../ldap/{name}` | 405 | 204 |
| `POST .../ldap/change-order` | solo por id | 204 resolviendo por nombre |
| `POST service/extdirect` `ldap_LdapServer.readTemplates` | 500 | 200 con las 4 plantillas |
| `verifyConnection` / `verifyUserMapping` / `verifyLogin` | sin endpoint propio | `success:false` + mensaje legible cuando fallan (vía `Responses.error`) |

Detalles del componente Direct:

- Las credenciales cifradas se sustituyen por `PASSWORD_PLACEHOLDER` antes de devolverlas a la UI
  (mismo patrón que la capability OIDC).
- `verifyLogin` decodifica la contraseña del navegador en base64/ISO-8859-1 (la UI envía
  `btoa(...)`), para no perder acentos.
- El mapeo XO↔config está centralizado en **`LdapServerXoMapper`**, compartido por el recurso REST
  y el componente Direct (evita la duplicación que producía el desfase de `order` al nulo).
- `clearCache` es un **no-op documentado que devuelve éxito**: el realm ejecuta
  `setCachingEnabled(false)` y `LdapDirectory` no guarda estado entre peticiones.

## 3. Cómo probarlo

```bash
# $NEXUS_PASSWORD: valor local en nexus-public/admin-creds.txt (ignorado por git)
export ADMIN="admin:$NEXUS_PASSWORD"; BASE=http://localhost:8081

# Plantillas del asistente (Ext.Direct)
curl -s -u "$ADMIN" -H 'Content-Type: application/json' \
  -X POST "$BASE/service/extdirect" -d '{
    "action":"ldap_LdapServer","method":"readTemplates","type":"rpc","tid":1,"data":[]
  }' | head -c 400

# CRUD por nombre
curl -s -u "$ADMIN" -X POST "$BASE/service/rest/v1/security/ldap" \
  -H 'Content-Type: application/json' -d '{ ... }'      # crear
curl -s -u "$ADMIN" "$BASE/service/rest/v1/security/ldap/mi-servidor"   # 200/404
curl -s -u "$ADMIN" -X DELETE "$BASE/service/rest/v1/security/ldap/mi-servidor"  # 204
```

Privilegios comprobables en la UI: **Administration → Security → Roles** → `nx-ldap-*`.

## 4. Tests y regresiones

- Java: `nexus-core` 368 y `nexus-coreui-plugin` 164 en verde.
- Frontend: 43 tests de Jest del módulo LDAP en verde (`yarn check-format` limpio).
- Los fallos de Jest que quedan en `SettingsSidebar`, `RepositoriesForm` y `GlobalHeader` son
  **preexistentes** (verificados con `git stash`) y no guardan relación con LDAP.

## 5. Límites conocidos

- Solo un directorio configurado a la vez (la capability `ldap` es singleton), igual que en la
  edición pública original.
- No hay pruebas automatizadas contra un servidor LDAP real; la verificación fue manual con
  `verifyConnection`/`verifyUserMapping`/`verifyLogin` contra un directorio de prueba.
- El *follow referrals* y el pool de conexiones usan los valores por defecto del conector
  (`unlimited`/`system default`); ajustarlos si el directorio los requiere.
