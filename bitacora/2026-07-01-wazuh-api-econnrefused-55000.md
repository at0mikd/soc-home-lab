# Bitácora — 2026-07-01 — Wazuh OVA: dashboard carga pero la API da ECONNREFUSED 55000

**Objetivo del día:** empezar el componente 1. Enrolar el `wazuh-agent` en la VM
`ubuntu-agent` contra el manager y confirmar que reporta. Ni llegué a eso.

**Qué intenté:**
1. Abrí el dashboard en `https://192.168.56.101` desde el host. La web carga —login
   incluido— así que di por hecho que el stack estaba arriba.
2. Adentro, el dashboard no mostraba datos y tiraba error de conexión con la VM.
3. Corrí `sudo /var/ossec/bin/wazuh-control status` en la VM `wazuh`: **todos** los
   daemons del manager en `not running`.
4. Ahí caí en que `wazuh-control status` solo mira el manager. El stack son tres
   servicios systemd separados: `wazuh-indexer` → `wazuh-manager` → `wazuh-dashboard`.
   El dashboard estaba vivo (por eso cargaba la web); el manager y su API, no.
5. `systemctl is-active` en los tres confirmó que el manager estaba caído.

**Qué rompió:**
```
connect ECONNREFUSED 127.0.0.1:55000
```
El dashboard busca la API del manager en localhost:55000. El `127.0.0.1` me despistó
un rato —pensé en un problema de red o de bind address— pero es el comportamiento
normal del OVA all-in-one: dashboard y API viven en la misma VM. El `ECONNREFUSED` no
era de red: no había proceso escuchando en 55000 porque el manager estaba muerto.

**Cómo lo resolví:**
Levanté los servicios en orden, dándole aire al indexer antes del manager:
```bash
sudo systemctl start wazuh-indexer
sleep 45
sudo systemctl start wazuh-manager
sudo systemctl start wazuh-dashboard
```
Después de eso, `wazuh-control status` mostró los daemons arriba y `ss -tlnp | grep
55000` devolvió algo escuchando. El dashboard dejó de tirar el error.

La causa de fondo: **el OVA no deja los servicios habilitados para auto-arranque.**
Arrancó bien la primera vez tras el import, pero en el siguiente boot quedaron
`inactive`. Lo cerré con:
```bash
sudo systemctl enable wazuh-indexer wazuh-manager wazuh-dashboard
```
`systemctl is-enabled` en los tres ya devuelve `enabled` y `ss -tlnp` confirma 55000
escuchando.

**Qué aprendí:**
- Que el dashboard cargue NO significa que el SIEM funcione. Son servicios separados;
  la web puede estar arriba con el manager muerto. Hay que mirar los tres.
- `wazuh-control status` ≠ estado del stack. Solo habla del manager. Para el cuadro
  completo, `systemctl is-active wazuh-indexer wazuh-manager wazuh-dashboard`.
- `ECONNREFUSED` a un `127.0.0.1` no es un problema de red: es "no hay nadie
  escuchando en ese puerto". Antes de tocar firewall o binds, verificar que el
  proceso esté vivo.

**Límite / deuda conocida:**
`enable` no garantiza el orden de arranque al boot. Si en un reboot el manager
levanta antes de que el indexer esté listo, puede volver a caerse con el mismo
síntoma. Queda pendiente verificar si la unit del manager ya declara
`After=wazuh-indexer.service`; si no, el workaround es un restart manual del manager
tras el boot. No lo toqué todavía.

**Pendiente para mañana:**
- Confirmar el `After=` del `wazuh-manager` y decidir si hace falta reforzar el orden.
- Recién ahí, arrancar el enrolamiento del agente en `ubuntu-agent`:
  versión de Ubuntu (journald vs `/var/log/auth.log`), IP host-only, instalar y
  registrar el agente contra `192.168.56.101`.
