
# Diseño Plugin Yum/RPM (F4)

## Estado
- F4a Hosted: implementado
- F4b Proxy: implementado  
- F4c Group: pendiente

## Arquitectura
- Formato: `yum` (`YumFormat`)
- Módulo: `public/common/components/formats/nexus-repository-yum`
- Facets: `YumContentFacet` (put/get/delete/putRpm/putGenerated), `YumMetadataFacet` (rebuild/ensureMetadata)
- Recipe: `yum-hosted`, `yum-proxy`
- Parser: `RpmHeaderParser` + modelos `RpmPackage`/`RpmDependency`/`RpmChangelog`
- Metadata: `YumMetadataBuilder` genera repomd + primary/filelists/other gzip

## Comportamiento
- Hosted PUT `*.rpm`: parse header, crea componente (namespace=arch, name, version=epoch:ver-rel), guarda asset, reconstruye repodata.
- GET `repodata/repomd.xml` (y demás): genera/actualiza si necesario, sirve desde assets generados.
- Proxy: pasa peticiones al upstream con caché; metadata repomd/primary se sirven desde caché.
- Rutas `repodata/*` no subibles/borrables manualmente.

## Almacenamiento
- Tablas `yum_*` vía `YumStoresProvider` + DAOs (auto-creadas)
- Metadatos generados en componente `_yum_metadata` (kind `yum-metadata`)
- RPMs en componentes `rpm` (kind `rpm`)

## REST
- V1/Beta: `POST/PUT /service/rest/v1/repositories/yum/hosted` y `yum/proxy` con atributos `yum:{}`

## Validación
Probado con Nexus corriendo: creación de repos, upload RPM real, descarga repodata/primary.
