# PostgreSQL como datastore de Nexus

> Estado: **soportado y verificado**. Nexus arranca contra PostgreSQL 15 y crea el esquema
> completo (86 tablas en la prueba). El datastore por defecto sigue siendo H2 embebido.

## 1. Qué se cambió en el fork

1. El driver `org.postgresql:postgresql` se añadió como dependencia `runtime` de la assembly
   (`public/selfhosted/assemblies/nexus-repository-core/pom.xml`), por lo que queda dentro de
   `BOOT-INF/lib` del uberjar. Antes el módulo MyBatis lo declaraba `optional` y no se empaquetaba.
2. Se añadió `DataStoreConfigurationEnvironmentSource` en `nexus-datastore-h2`: lee la
   configuración del datastore desde variables de entorno o propiedades de sistema, con prioridad
   sobre los valores embebidos.
3. `nexus-datastore-mybatis` ya traía el resto: detección de `jdbc:postgresql`, driver, HikariCP y
   SQL específico por motor (`databaseId="PostgreSQL"`).

No hay código propietario implicado: todo es EPL-1.0 del árbol público.

## 2. Variables de entorno

El nombre del datastore es `nexus`; el patrón es `NEXUS_DATASTORE_<STORE>_<ATRIBUTO>`:

| Variable | Equivalente de sistema | Ejemplo |
|---|---|---|
| `NEXUS_DATASTORE_NEXUS_JDBCURL` | `nexus.datastore.nexus.jdbcUrl` | `jdbc:postgresql://127.0.0.1:5434/nexus` |
| `NEXUS_DATASTORE_NEXUS_USERNAME` | `nexus.datastore.nexus.username` | `nexus` |
| `NEXUS_DATASTORE_NEXUS_PASSWORD` | `nexus.datastore.nexus.password` | `nexus` |
| `NEXUS_DATASTORE_NEXUS_SCHEMA` | `nexus.datastore.nexus.schema` | `public` |
| `NEXUS_DATASTORE_NEXUS_ADVANCED` | `nexus.datastore.nexus.advanced` | `maximumPoolSize=20` |
| `NEXUS_DATASTORE_NEXUS_TYPE` | `nexus.datastore.nexus.type` | `jdbc` (por defecto) |

- Las variables de entorno ganan a las propiedades de sistema.
- Si no hay `jdbcUrl` que empiece por `jdbc:`, la fuente no aplica y Nexus usa H2.
- Para un datastore con otro nombre, sustituir `NEXUS` por el nombre en mayúsculas.

## 3. Levantar PostgreSQL (ejemplo con Docker)

No hace falta `sudo` si el usuario está en el grupo `docker`. En la prueba se usó el puerto
**5434** para no chocar con otros contenedores del host.

```bash
docker run -d --name nexus-pg \
  -e POSTGRES_USER=nexus \
  -e POSTGRES_PASSWORD=nexus \
  -e POSTGRES_DB=nexus \
  -p 127.0.0.1:5434:5432 \
  postgres:15-alpine

# esperar a que acepte conexiones
until docker exec nexus-pg pg_isready -U nexus >/dev/null 2>&1; do sleep 1; done
docker exec nexus-pg psql -U nexus -d nexus -c 'select version();'
```

Los datos persisten en el volumen del contenedor. Para empezar de cero:

```bash
docker rm -f nexus-pg
docker volume rm $(docker inspect -f '{{range .Mounts}}{{.Name}}{{end}}' nexus-pg)  # si aplica
```

## 4. Arrancar Nexus contra PostgreSQL

Usar un directorio de trabajo **separado** evita mezclar la base H2 existente con PostgreSQL:

```bash
cd nexus-public/public/selfhosted/assemblies/nexus-repository-core/target/assembly
export JAVA_HOME=/home/ladmin/Downloads/nexus-fork/toolchain/jdk-21.0.13+9
export PATH=$JAVA_HOME/bin:$PATH

export KARAF_DATA=/tmp/opencode/nexus-pg-work
mkdir -p "$KARAF_DATA/etc"
printf 'application-port=8082\napplication-host=127.0.0.1\n' > "$KARAF_DATA/etc/nexus.properties"

export NEXUS_DATASTORE_NEXUS_JDBCURL='jdbc:postgresql://127.0.0.1:5434/nexus'
export NEXUS_DATASTORE_NEXUS_USERNAME=nexus
export NEXUS_DATASTORE_NEXUS_PASSWORD=nexus
export NEXUS_DATASTORE_NEXUS_SCHEMA=public

./bin/nexus run
```

- `KARAF_DATA` cambia `sonatype-work/nexus3` por el directorio indicado (logs, blobs, etc.).
- El `admin.password` inicial se genera en `$KARAF_DATA/admin.password`.
- Para el puerto 8081 normal, no definir `application-port` y usar el `KARAF_DATA` por defecto.

## 5. Verificación

```bash
# 1) Nexus responde
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:8082/service/rest/v1/status   # 200

# 2) El esquema existe en PostgreSQL
docker exec nexus-pg psql -U nexus -d nexus -tAc \
  "select count(*) from information_schema.tables where table_schema='public';"          # 86

docker exec nexus-pg psql -U nexus -d nexus -tAc \
  "select table_name from information_schema.tables where table_schema='public' order by 1 limit 10;"

# 3) MyBatis eligió el SQL de PostgreSQL (buscar en el log)
grep -i 'databaseId\|PostgreSQL' /tmp/opencode/nexus-pg.log | head
```

Comprobaciones realizadas en la prueba:

- `MyBatis databaseId: PostgreSQL` en el arranque.
- 86 tablas creadas (`repository`, `npm_component`, `npm_asset`, `api_key`, etc.).
- `GET /service/rest/v1/status` → 200 y autenticación con el `admin.password` generado.
- Los repositorios y datos de la instancia H2 **no** aparecen: es una base nueva.

## 6. Migración de datos H2 → PostgreSQL

No está incluida. Opciones:

1. **Recrear** repositorios y volver a publicar/proxy-cachear el contenido (lo más simple y
   verificable).
2. Exportar/importar tabla por tabla (`repository`, `*_component`, `*_asset`, `*_asset_blob`, …)
   respetando claves y blobs. No es un proceso soportado por este fork; requiere script propio y
   validación de integridad.
3. Mantener H2 para la instancia existente y usar PostgreSQL solo para instalaciones nuevas.

En todos los casos, los **blobs** viven en `sonatype-work/nexus3/blobs` (o el blob store
configurado), no en PostgreSQL: hay que copiarlos aparte.

## 7. Volver a H2

Basta con no definir `NEXUS_DATASTORE_NEXUS_JDBCURL` y arrancar con el `KARAF_DATA` original. La
base H2 sigue intacta en `sonatype-work/nexus3/db`.

## 8. Notas de producción

- Fijar `maximumPoolSize` en `NEXUS_DATASTORE_NEXUS_ADVANCED` según la carga y el límite de
  conexiones del servidor.
- Hacer backup lógico (`pg_dump`) además de los blobs; ambos deben ser consistentes entre sí.
- El cifrado de secretos de MyBatis usa `nexus.mybatis.cipher.password` (por defecto `changeme`).
  Cambiarlo **antes** de la primera escritura de secretos; si se cambia después, los valores
  existentes no se pueden descifrar.
- Para HA se necesita además una estrategia de bloqueo/tareas distribuidas; ver
  `docs/FUNCIONALIDADES_PRO.md`.
