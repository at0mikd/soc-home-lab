# Bitácora — 2026-08-05 — Levantar el stack Wazuh en Docker (INFRA-5 a INFRA-6)

**Objetivo del día:** bajar el `docker-compose.yml` que arma el lab y dejar
los cuatro servicios healthy. Resultado: lo logré, pero el camino tuvo tres
problemas de los que nadie documenta y que vale la pena dejar escritos.

**Estado final del día:** indexer, manager, dashboard y víctima `healthy`.
El dashboard responde en `https://localhost:5601`. El agente de la víctima
queda para confirmar en el próximo turno (INFRA-7).

---

## El problema que NO era: el certificado

El primer síntoma fue un error de TLS en el manager:

```
x509: certificate is not valid for any names, but wanted to match wazuh.indexer
```

Pensé que era un problema de CA o de que el filebeat usaba el cert equivocado.
Verifiqué fingerprints, miré el `openssl s_client`... la CA estaba bien, los
certs verificaban contra `root-ca.pem`. Resultado real:

**El `wazuh-certs-tool` me había generado los certs con SAN de IP, no de
hostname.** En `conf/certs.yml` yo le había pasado `ip: 172.25.0.20` (una IP
real), así que el SAN salió `IP:172.25.0.20`. Pero el filebeat conecta por
nombre (`https://wazuh.indexer:9200`) y Go x509 ya no usa el CN para validar
hostname, exige SAN. Por eso fallaba.

**Fix:** reescribí `conf/certs.yml` para usar hostnames como valores de `ip:`
(igual que el docker oficial), regeneré y el SAN pasó a `DNS:wazuh.indexer`.
Lección: el tool no hace magia, mete en el SAN literalmente lo que le das.

## El problema que SÍ era un hallazgo: securityadmin

Después de arreglar el cert, el manager pasó a otra etapa:

```
503 Service Unavailable: OpenSearch Security not initialized
```

Esto no venía en ningún tutorial que yo haya leído. Abrí `/entrypoint.sh` de la
imagen `wazuh/wazuh-indexer:4.14.5` y el bloque que ejecuta `securityadmin`
está **comentado**:

```bash
#if [[ "$DISCOVERY" == "single-node" ]] && [[ ! -f "/var/lib/wazuh-indexer/.flag" ]]; then
  # run securityadmin.sh for single node...
#fi
```

O sea: en el primer arranque el indexer queda con OpenSearch Security sin
inicializar y rechaza toda llamada autenticada. Lo corrí a mano:

```bash
docker exec -u root wazuh-indexer \
  bash -c 'JAVA_HOME=/usr/share/wazuh-indexer/jdk \
  /usr/share/wazuh-indexer/plugins/opensearch-security/tools/securityadmin.sh \
  -cd /usr/share/wazuh-indexer/config/opensearch-security \
  -cacert /usr/share/wazuh-indexer/config/certs/root-ca.pem \
  -cert /usr/share/wazuh-indexer/config/certs/admin.pem \
  -key /usr/share/wazuh-indexer/config/certs/admin-key.pem \
  -h 127.0.0.1 -p 9200 -icl -nhnv'
```

`Done with success`, y el cluster pasó a `green`. Documentado en el
`setup/README.md` como paso obligatorio del primer boot.

Nota para el futuro: el config de security vive en el volumen
`wazuh_indexer_data`, así que esto solo hace falta en un arranque con el
volumen vacío. Pero como el entrypoint está comentado, si algún día se borra el
volumen hay que volver a correrlo.

## Los healthchecks que mentían

Después de activar security, el indexer quedó `health: starting` para siempre.
El healthcheck hacía `curl` al endpoint de salud **sin credenciales** → recibía
`Unauthorized` → nunca healthy. Lo corregí agregando `-u admin:SecretPassword`.

El dashboard tenía el problema gemelo: el `GET /status` devuelve `401` cuando
security está encendido, y el healthcheck esperaba `green|yellow|red`. Lo cambié
para que solo verifique que el puerto responda.

Y el dashboard además falló una vez con una migración `.kibana_1` a medias (el
host lento hizo que el request al indexer agotara el timeout) y quedó diciendo
"otra instancia está migrando". Lo resolví borrando el índice vacío y
reiniciando el dashboard.

## El problema que no es de código: recursos

El host tiene 15 GB de RAM. Docker Desktop reserva ~3.7 GB para su VM y el
stack en caliente usa ~2.2 GB. Con Brave y VS Code abiertos el host se fue a
swap y **todos** los comandos Docker (`up`, `down`, `stop`) colgaban. Los
`process_cluster_event_timeout_exception` del indexer eran pura falta de RAM,
no un bug de config.

Dejé los números reales en el README: `docker stats` mostró indexer ~1.4 GB,
manager ~600 MB, dashboard ~200 MB, víctima ~10 MB.

## El puerto 443 que "no se veía"

El dashboard respondía con `curl` local pero no desde el navegador. El culpable:
el `firewalld` por defecto de Fedora solo abre `1025-65535`, y el dashboard
estaba publicado en `443`. El `curl` local pasaba porque el tráfico a la propia
IP va por loopback. Lo moví a `5601` (dentro del rango) y quedó accesible.

---

## Qué falta para cerrar INFRA-6 de verdad

- Confirmar INFRA-7 (que el agente de la víctima se enroló). Aún no lo verifiqué.
- Los `screenshots/` los dejé sin commitear: contienen la sesión de browser y no
  pude revisarlos por mi cuenta. Hay que tapar hostnames/IPs antes de subirlos.
