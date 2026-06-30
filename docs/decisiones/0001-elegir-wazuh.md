# ADR 01: Wazuh como SIEM del lab

**Fecha:** 2026-06-30 · **Estado:** aceptada

## Contexto

Necesito un SIEM para el lab: algo que ingiera logs de varios endpoints, los parsee,
corra reglas de detección y me muestre alertas. Restricciones reales:

- Presupuesto cero. Tiene que ser open source o free tier.
- Corre en una notebook/PC con recursos justos (4-6 GB para el SIEM, no más).
- Que aporte a un CV de SOC Jr: las ofertas piden SIEM, y nombran Wazuh, Splunk o ELK.
- Que traiga detección y mapeo a MITRE ATT&CK sin que tenga que armar todo de cero.

## Opciones consideradas

1. **Wazuh** — open source, agente liviano, trae reglas y decoders out-of-the-box,
   mapeo a ATT&CK incluido, FIM y respuesta activa. Contra: el stack completo (manager +
   indexer + dashboard) come RAM; la curva de las reglas XML no es trivial.
2. **ELK + ElastAlert** — flexibilísimo y muy pedido en el mercado. Contra: hay que
   armar la detección casi entera a mano (no viene con reglas), más piezas que mantener,
   más pesado. Mucha plomería antes de ver una alerta.
3. **Security Onion** — muy completo (NSM + Suricata + Zeek + Elastic). Contra: pensado
   para sensor de red dedicado; los requisitos de hardware se van por encima de lo que
   tengo. Sobredimensionado para empezar.
4. **Splunk Free** — estándar de la industria, gran experiencia. Contra: la licencia
   free topea en 500 MB/día de ingesta y recorta features clave (alerting, auth). El
   límite molesta justo cuando querés generar tráfico de ataque.

## Decisión

Wazuh, en instalación all-in-one sobre una VM.

## Por qué

Es el que me deja ver una alarma antes —trae detección y ATT&CK de fábrica— sin pagar
ni pelear con el límite de ingesta de Splunk. Aparece en las ofertas de SOC Jr que estoy
mirando, así que la práctica es transferible. ELK lo descarté no por malo sino por
costo de arranque: quiero detectar, no pasarme la primera semana cableando ElastAlert.
Security Onion y Splunk los descarté por hardware y por el techo de licencia,
respectivamente.

## Consecuencias

- **Gano**: alertas y mapeo ATT&CK rápido, agente liviano para sumar endpoints (incluido
  el de Azure y, más adelante, contenedores).
- **Resigno**: la experiencia directa con Splunk/ELK, que también piden. Queda como deuda
  para una etapa posterior si el tiempo da.
- **Deuda**: el stack consume RAM; si la notebook sufre, hay que evaluar separar el
  indexer o bajar retención. Las reglas XML las voy a tener que aprender sobre la marcha.
