# ADR 0001: Choose Wazuh as the lab SIEM

- **Date:** 2026-06-30
- **Status:** accepted

## Context

I need a SIEM for the lab: something that ingests logs from several endpoints,
parses them, runs detection rules, and shows me alerts. Real constraints:

- Zero budget. Open source or free tier.
- Runs on a laptop with tight resources (4-6 GB available for the SIEM).
- Helps a SOC Jr resume: job postings ask for SIEM, naming Wazuh, Splunk, or ELK.
- Comes with detection and MITRE ATT&CK mapping out of the box, so I do not
  have to build it from scratch.

## Options considered

1. **Wazuh** — open source, lightweight agent, ships with rules and decoders,
   ATT&CK mapping included, FIM and active response. Downside: the full stack
   (manager + indexer + dashboard) eats RAM; the XML rules are not trivial.
2. **ELK + ElastAlert** — extremely flexible and very popular. Downside: the
   detection side has to be built almost from scratch (no rules out of the
   box), more moving parts, heavier. A lot of plumbing before the first alert.
3. **Security Onion** — very complete (NSM + Suricata + Zeek + Elastic).
   Downside: designed for a dedicated network sensor; hardware requirements
   go well beyond what I have. Overkill for a starting point.
4. **Splunk Free** — industry standard, great experience. Downside: the free
   license caps at 500 MB/day of ingest and cuts key features (alerting,
   auth). The cap bites exactly when you want to generate attack traffic.

## Decision

Wazuh, all-in-one on a single VM (later replaced by Docker — see ADR 0002).

## Why

It lets me see an alarm sooner — detection and ATT&CK out of the box — without
paying or fighting Splunk's ingest cap. Wazuh shows up in the SOC Jr postings
I am looking at, so the practice transfers. I dropped ELK not because it is
bad but because of the startup cost: I want to detect, not spend the first
week wiring ElastAlert. Security Onion and Splunk were dropped for hardware
and for the license cap, respectively.

## Consequences

- **Gain:** alerts and ATT&CK mapping fast, lightweight agent to add
  endpoints (including the future Azure one and, later, containers).
- **Trade-off:** direct experience with Splunk/ELK, which are also asked for.
  Left as debt for a later phase if time allows.
- **Debt:** the stack consumes RAM; if the laptop suffers, the indexer will be
  split out or retention lowered. The XML rules will be learned on the job.
