# soc-home-lab — Home lab de SOC

Armo un home lab de SOC y documento el proceso en público: qué monté, qué rompí y cómo
lo detecté.

Base: fundamentos de SOC (SIEM, detección, respuesta) sobre Wazuh. Diferenciador:
monitoreo de **infraestructura moderna** — containers, Kubernetes y pipelines de CI/CD.

## Roadmap

| # | Componente | Competencia SOC | Estado |
|---|---|---|---|
| 1 | Wazuh base + detección SSH brute-force | SIEM, detección, ATT&CK | en curso |
| 2 | Hardening del endpoint (SSH, ufw, fail2ban) | Linux / hardening | planeado |
| 3 | Detección de escalada/persistencia local | Detección / ATT&CK | planeado |
| 4 | Respuesta a incidente (caso simulado) | Respuesta a incidentes | planeado |
| 5 | SOAR: Wazuh → n8n → Discord | Automatización / SOAR | planeado |
| 6 | Threat intel: IOCs + correlación | Threat intelligence | planeado |
| 7 | Docker host monitoring | Containers (diferenciador) | planeado |
| 8 | Kubernetes monitoring | K8s (diferenciador) | planeado |
| 9 | CI/CD: pipeline Jenkins + monitoreo | CI-CD (diferenciador) | planeado |
| 10 | Endpoint en Azure reportando al SIEM | Nube | planeado |
| 11 | Anexo: writeup de hallazgo IDOR (bug bounty) | — | planeado |

## Arquitectura (fase local)

```
  Host
   ├── VM "siem"     → Wazuh (manager + indexer + dashboard)
   └── VM "victima"  → wazuh-agent + sshd, en red interna
   (atacante = el host, con hydra contra la VM víctima)
```

## Cómo está organizado el repo

- `docs/decisiones/` — ADRs: por qué elegí cada cosa y qué descarté.
- `docs/writeups/` — detecciones e incidentes, con mapeo a MITRE ATT&CK (se crea con el primer writeup).
- `bitacora/` — notas crudas del día: qué intenté, qué rompió (se crea con la primera entrada).
- `setup/`, `wazuh-config/`, `scripts/` — configs y guías reproducibles del SIEM.
- `incident-reports/` — registros de respuesta a incidentes.
- `architecture/`, `docker/`, `kubernetes/`, `jenkins/` — estructura para los
  componentes del diferenciador (vacías por ahora).
