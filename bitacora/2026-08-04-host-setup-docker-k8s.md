# Bitácora — 2026-08-04 — INFRA-1 a INFRA-3: dejar el host listo para Docker y K8s

**Objetivo del día:** terminar las tres tareas de host que necesita el lab
Docker. La sesión real con los contenedores de Wazuh queda para el próximo
turno.

**Lo que tenía que instalar (y registrar) en esta sesión:**

- INFRA-1: verificar Docker.
- INFRA-2: instalar `kubectl`.
- INFRA-3: instalar `minikube` y probar `minikube start`.

---

## INFRA-1 — Verificar Docker

Asumía que era un Docker nativo de Fedora, pero el `docker info` cantó otra cosa:

```
 Server Version: 29.6.2
 Storage Driver: overlayfs
 Cgroup Version: 2
 Kernel Version: 6.12.76-linuxkit
 Operating System: Docker Desktop
```

Es **Docker Desktop** sobre `linuxkit`. Esto explica dos cosas raras del
reconocimiento inicial:

- `systemctl is-active docker` daba `failed`. Docker Desktop no corre como
  servicio systemd, corre su propio daemon fuera de systemd. El socket
  `/var/run/docker.sock` igual existe y responde, así que el lab funciona.
- El driver Docker de `minikube` lo noté a tiempo y va a ser relevante más
  abajo.

**Comandos usados:**

```bash
docker info | grep -E 'Server Version|Storage Driver|Cgroup Version|Kernel Version|Operating System'
docker run --rm hello-world   # smoke test: el motor arranca contenedores reales
```

**Resultado:** OK. `docker run --rm hello-world` baja la imagen, crea el
contenedor y muestra el mensaje de bienvenida. Daemon sano.

---

## INFRA-2 — Instalar `kubectl`

No estaba instalado (`which kubectl` vacío). Descarté `dnf install
kubernetes-client` porque la versión de los repos de Fedora 44 suele ir
atrasada y la idea es tener la última estable sin esperar. Fui por el binario
oficial desde `dl.k8s.io`.

**Comandos usados:**

```bash
curl -fsSL -o /tmp/opencode/kubectl \
  "https://dl.k8s.io/release/$(curl -sL https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x /tmp/opencode/kubectl
mv /tmp/opencode/kubectl /home/tomas/.local/bin/kubectl   # PATH de usuario, sin sudo
kubectl version --client
```

**Versión:** `Client Version: v1.36.3`, `Kustomize Version: v5.8.1` (embebido).
Importante para el roadmap: Kustomize ya viene con el binario, así que para
el componente #8 (K8s monitoring) no hace falta un `helm` extra. Decisión
que ya estaba tomada en el ADR 0002 ("Kustomize + minikube"), confirmada
acá por la disponibilidad.

**Piedra chica:** el primer intento de `mv` fue a `/usr/local/bin` con
`sudo`. La sesión actual no tiene TTY, así que `sudo` quedó colgado pidiendo
password. Solución: PATH de usuario (`~/.local/bin`), sin sudo. Anoto para
el README: en una sesión interactiva real, el operador probablemente prefiera
`sudo install` en `/usr/local/bin` para tener la CLI accesible a todos los
usuarios. La dejo donde está hasta que sepamos si esto lo va a tocar alguien
más.

---

## INFRA-3 — Instalar `minikube`

Idem: binario oficial desde GitHub. Verifiqué antes la doc — confirmé que
"on Linux, the Docker driver does not require virtualization to be enabled",
lo que importa porque este host corre Docker Desktop y no tiene
virtualización "real".

**Comandos usados:**

```bash
curl -fsSL -o /tmp/opencode/minikube-linux-amd64 \
  https://github.com/kubernetes/minikube/releases/latest/download/minikube-linux-amd64
chmod +x /tmp/opencode/minikube-linux-amd64
mv /tmp/opencode/minikube-linux-amd64 /home/tomas/.local/bin/minikube
minikube version
minikube config set driver docker
```

**Versión:** `minikube v1.38.1`.

**Smoke test: `minikube start`.** No salió limpio la primera vez:

1. Primer intento con `timeout 300`: se quedó colgado en "Pulling base image
   v0.0.50" sin avanzar. Posiblemente porque Docker Desktop tiene un pull
   inicial lento y timeout corto. No quedó ningún contenedor ni proceso
   minikube vivo.
2. `minikube delete` (limpio el estado) y segundo intento con `timeout 600`:
   terminó OK. Mensajes clave:

   ```
   * Using the docker driver based on existing profile
   * Starting "minikube" primary control-plane node in "minikube" cluster
   * Pulling base image v0.0.50 ...
   * docker "minikube" container is missing, will recreate.
   * Configuring bridge CNI (Container Networking Interface) ...
   * Verifying Kubernetes components...
     - Using image gcr.io/k8s-minikube/storage-provisioner:v5
   * Enabled addons: storage-provisioner, default-storageclass
   * Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
   ```

   `kubectl get nodes` devolvió:

   ```
   NAME       STATUS   ROLES           AGE    VERSION
   minikube   Ready    control-plane   101s   v1.35.1
   ```

**Dos warnings que vale la pena registrar:**

- `! For an improved experience it's recommended to use Docker Engine
  instead of Docker Desktop.` — minikube me lo dice explícito. No es bloqueante
  pero confirma que la elección de Docker Desktop tiene un costo en este
  lab. Anotado en la sección "deuda" del ADR 0002.
- `! Starting v1.39.0, minikube will default to "containerd" container
  runtime.` — el runtime por defecto de minikube va a cambiar. Cuando
  actualicemos `minikube` (>1.39), hay que tener esto presente para que
  Wazuh siga corriendo como se espera dentro del cluster.

**Cierre: `minikube stop`.** Apago el cluster para liberar ~2 GB de RAM
antes de la siguiente sesión. El binario queda instalado y el perfil
queda configurado con `driver=docker` para el próximo `start`. Cuando
lleguemos al componente #8 (K8s monitoring) retomamos desde acá.

```bash
minikube stop
docker ps -a --filter 'name=minikube'
# minikube  Exited (130) 2 seconds ago
```

---

## Estado al cierre

| Item | Estado | Ubicación |
|---|---|---|
| Docker daemon | Sano, Docker Desktop 29.6.2 sobre linuxkit | `/var/run/docker.sock` |
| Docker Compose | v5.3.1 (plugin v2) | `/usr/local/bin/docker` |
| `kubectl` | v1.36.3 + Kustomize v5.8.1 | `/home/tomas/.local/bin/kubectl` |
| `minikube` | v1.38.1, driver=docker configurado, cluster apagado | `/home/tomas/.local/bin/minikube` |
| Cluster K8s de prueba | Apagado (liberó ~2 GB) | contenedor `minikube` en `Exited` |

---

## Verificación end-to-end (misma sesión): dashboard accesible

Después del cierre inicial, el operador pidió que confirme que el dashboard de
Kubernetes realmente se podía alcanzar desde este Fedora 44. No era un smoke
test más: era "ya entraste a la UI y viste los pods corriendo".

**Lo que hice:**

1. `minikube start --driver=docker` — el primer intento falló con
   `PROVIDER_DOCKER_NOT_RUNNING: deadline exceeded running "docker version
   --format <no value>-<no value>:<no value>": signal: killed`. El daemon
   estaba lento respondiendo. Un par de segundos después, `docker version` y
   `docker ps` volvieron a responder rápido (sin tocar nada), así que fue un
   hipo transitorio, no una falla real. Reintenté `minikube start` y anduvo.
2. `minikube addons enable dashboard` + `minikube addons enable metrics-server`.
3. Esperé a que los pods pasaran de `ContainerCreating` a `Running` (unos 80 s
   en este host, lo dejo apuntado).
4. Levanté `kubectl proxy --address=127.0.0.1 --port=8001 --accept-hosts='^127\.0\.0\.1$,^localhost$'`
   desacoplado del shell (`nohup` + `&`) y guardé el PID 207717.
5. Verifiqué con `curl -sS -w '%{http_code}'` que la ruta del dashboard
   respondía HTTP 200.
6. Creé un ServiceAccount `admin-user` con `cluster-admin` y le generé un
   bearer token con `kubectl -n kubernetes-dashboard create token admin-user`.
   El token quedó en `/tmp/opencode/dashboard-token.txt` (fuera del repo).
7. El operador abrió `http://127.0.0.1:8001/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/`
   en el browser, se logueó con el token y confirmó que en el namespace
   `kubernetes-dashboard` veía los pods `dashboard` y `metrics-scraper` en
   `Running`. Confirmado de punta a punta.
8. `kill 207717` y `minikube stop` al cierre.

**Lo que aprendí en esta segunda pasada:**

- El `PROVIDER_DOCKER_NOT_RUNNING` que tira minikube a veces es **transitorio**
  en Docker Desktop. No salir corriendo a hacer `docker system prune` ni
  reiniciar el daemon; probar dos veces con un par de segundos entre intento.
- El addon `dashboard` de minikube **no crea un ServiceAccount con permisos**
  por defecto. Para entrar a la UI con permisos de admin hay que crear
  `admin-user` + `ClusterRoleBinding cluster-admin` + token manualmente.
  Lo dejo como comando reproducible arriba; cuando lo armemos en limpio,
  esto va al `setup/` como un script `bootstrap-dashboard.sh`.
- `kubectl proxy` + `nohup` es el camino correcto para exponer el dashboard
  al host sin abrir el puerto a la red. `--accept-hosts` con la regex de
  `127.0.0.1` / `localhost` evita que un servicio en otra IP del mismo host
  pueda pegarle al proxy.

**Riesgos y notas para futuras sesiones:**

- El token del dashboard dura 1 hora por defecto (`exp: ...` en el JWT). Si
  la próxima sesión quiere entrar otra vez, hay que regenerarlo con el
  mismo comando. No hay refresh tokens en minikube.
- El proxy escucha solo en loopback. Para exponerlo a otra IP hay que
  re-lanzar con `--address=0.0.0.0` y aceptar el riesgo (no recomendado en
  una laptop en red pública).

**Pendiente para la próxima sesión (continúa con INFRA-4 en adelante):**

- INFRA-4: instalar `hydra` desde `dnf`.
- INFRA-5: generar los certs SSL del manager Wazuh dentro de
  `setup/conf/manager_ssl_certs/`.
- INFRA-6: `docker compose up -d` desde `setup/`, observar logs hasta que
  los tres servicios estén healthy.
- INFRA-7: verificar que el agente de la víctima enrola (`wazuh-agentd -t`).
- INFRA-8: generar fallos SSH a mano contra `localhost:2222` y confirmar las
  reglas 5710 / 5712 en el dashboard.
- INFRA-9: bitácora con todo lo que rompió en la sesión real (esta sólo
  cubrió host).
- INFRA-10: primer writeup, `docs/writeups/0001-ssh-brute-force.md`.

**Lo que aprendí en esta sesión:**

- Que `docker info` te dice cosas que el `systemctl is-active` miente. Si el
  socket responde y `docker run` baja una imagen, el daemon está sano. El
  servicio systemd puede estar `failed` y no importa.
- Que `minikube start` con Docker Desktop no es rápido. El primer pull de la
  base image lleva minutos. Pedir `timeout 600` para no cortar a la mitad.
- Que `Kustomize v5.8.1` ya viene con `kubectl v1.36.3`. Una pieza menos
  para pensar cuando llegue el componente K8s.
- Que `sudo` sin TTY se cuelga pidiendo password. Para herramientas del
  usuario, `~/.local/bin` (que ya está en PATH) es el camino limpio.