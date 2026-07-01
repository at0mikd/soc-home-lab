# Bitácora — 2026-07-01 — Enrolar el wazuh-agent en la víctima Ubuntu

**Objetivo del día:** con el manager ya arriba, instalar el `wazuh-agent` en la VM
`ubuntu-agent` y que reporte `Active` al manager `192.168.56.101`. Meta simple; el
camino tuvo cuatro piedras.

**Qué intenté (en orden):**
1. Quise entrar por SSH a la víctima para instalar. `systemctl status ssh` →
   `Unit ssh.service could not be found`. No tenía `openssh-server` instalado.
2. Fui a instalarlo con `apt-get`. Falló al resolver el mirror.
3. Arreglado eso, quise agregar el repo de Wazuh. El `gpg --import` se enredó y apt
   tiró `NO_PUBKEY`.
4. Resuelta la clave, instalé el agente. Reportó `Active`, pero con una IP distinta
   a la que yo creía.

**Qué rompió (los errores reales):**

sshd no instalado:
```
Unit ssh.service could not be found.
```

Sin internet en la víctima (está solo en host-only, sin NAT, por diseño):
```
Fallo temporal al resolver «ar.archive.ubuntu.com»
```

La clave GPG de Wazuh no quedó importada (el pipe a `gpg --import` chocó creando
`/root/.gnupg`):
```
gpg: Fatal: no se puede crear el directorio '/root/.gnupg': El archivo ya existe
...
Err:2 https://packages.wazuh.com/4.x/apt stable InRelease
  NO_PUBKEY 96B3EE5F29111145
```

**Cómo lo resolví:**

1. **sshd:** desde la consola de la VM, `sudo apt-get install -y openssh-server` +
   `systemctl enable --now ssh`. Ese mismo sshd es el blanco de hydra después, así
   que lo necesito sí o sí.

2. **Internet en la víctima:** la aislé a propósito (host-only, sin NAT), y sin NAT
   no hay repos. Chicken-and-egg. Le agregué un **Adaptador 2 NAT temporal** en
   VirtualBox (`enp0s8`, DHCP → `10.0.2.15`), instalé todo, y lo apago después para
   restaurar el aislamiento. Nota: `10.0.2.15` es la NAT interna de VirtualBox, NO
   se alcanza desde el host; al host se llega siempre por la host-only `192.168.56.x`.

3. **Clave GPG:** el método robusto es `--dearmor`, no `--import` por pipe:
   ```
   curl -s -o /tmp/wazuh.key https://packages.wazuh.com/key/GPG-KEY-WAZUH
   sudo gpg --batch --yes --dearmor -o /usr/share/keyrings/wazuh.gpg /tmp/wazuh.key
   ```
   Verifiqué con `gpg --show-keys` que el ID terminara en `96B3EE5F29111145` (el del
   error). Ahí apt dejó de quejarse.

4. **Instalación:** `sudo WAZUH_MANAGER="192.168.56.101" apt-get install -y
   wazuh-agent=4.14.5-1` (versión pineada para igualar al manager) + `enable --now`.
   El dashboard mostró el agente `001 ubuntu-agent-VirBox`, Ubuntu 22.04.5, `active`.

5. **La IP que bailó:** el dashboard reportó la víctima en `192.168.56.103`, no en
   `.102` como yo había anotado. La host-only estaba en DHCP. Para un lab de
   detección eso es veneno: si la IP de la víctima cambia sola, hydra pega al blanco
   equivocado y los logs quedan inconsistentes. La fijé estática por netplan:
   ```yaml
   network:
     version: 2
     renderer: networkd
     ethernets:
       enp0s3:
         dhcp4: no
         addresses: [192.168.56.103/24]
       enp0s8:
         dhcp4: yes
         optional: true
   ```
   Sin gateway ni DNS en la host-only (segmento aislado). `optional: true` en la NAT
   para que el boot no cuelgue cuando la apague.

**Qué aprendí:**
- El aislamiento tiene costo operativo: cada instalación en la víctima obliga a
  prender/apagar el NAT. Es el precio de tenerla limpia durante los ataques.
- `gpg --import` por pipe con sudo es frágil; `--dearmor` a keyring es el camino.
- Una víctima con IP por DHCP no sirve para un lab de detección reproducible. IP
  estática desde el día uno.

**Estado al cierre:** agente `Active`, IP fija en `192.168.56.103`, manager en
`192.168.56.101`. VMs apagadas.

**Pendiente para mañana:**
- Al prender, **apagar el Adaptador 2 (NAT)** y confirmar que el agente sigue
  `active` reportando solo por host-only. (No alcancé a verificarlo hoy.)
- Confirmar el `After=wazuh-indexer.service` del manager (deuda de la bitácora
  anterior).
- Generar **fallos SSH a mano** (`ssh usuariofalso@192.168.56.103`) y verificar en el
  dashboard que disparan 5710/5716 y, tras el umbral, la 5712 — antes de tocar hydra
  y de escribir la regla custom.
