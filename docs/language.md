# Language policy (public repo)

Everything that lands on GitHub is written in **English**. This includes:

- `README.md` and the rest of the public-facing docs at the repo root.
- `docs/decisions/` (ADRs), `docs/writeups/` and any future `docs/` content.
- `setup/README.md` and any file in `setup/` that is meant to be read by a human
  (`docker-compose.yml`, `Dockerfile.victima`, the YAML/TOML/JSON configs and
  their inline comments).
- Bitácoras and writeups that are tracked in git and therefore ship with the repo.
- Commit messages and PR descriptions.

**Out of scope (stay in Spanish):** `CLAUDE.md`, the `.claude/` directory, and the
`plantillas/` directory are listed in `.gitignore` and are not part of the public
repo. They are the operator's working notes and stay in Spanish for fluidity.

**Why English:** the repo is a portfolio piece. Reviewers, recruiters, and people
copying the lab are more likely to land here from an English-language search than
from a Spanish one, and most of the source material I draw on (Wazuh, ATT&CK,
Docker, Kubernetes, MITRE) is in English anyway. Bilingual docs would dilute
both languages.

**If a piece of content is easier to read in Spanish (e.g. a quick debugging
note),** it does not belong in the public repo. Put it in `bitacora/` only if
you are willing to translate it before committing, or keep it as a private note
outside the repo.
