# Posternity

**An out-of-band management enclave reference architecture for research computing.**

Every server in a research fleet carries a baseboard management controller — iDRAC, iLO, OpenBMC — a small always-on computer with total power over its host.  BMCs are chronically under-secured (default credentials, slow firmware, a protocol with known cryptographic defects) and chronically over-exposed (flat management VLANs reachable by whole departments, or worse).

Posternity is a complete answer: a bounded, segmented, zero-egress enclave that contains all BMC traffic, gates every human through one of two identity-backed doors, records every session, curates its own software library, and documents itself as a System Security Plan mapped to NIST SP 800-171.

> **Why "Posternity"?**  A *postern* is the small, guarded rear gate of a fortification — the private entrance used by those who belong there.  The reference architecture is also the durable artifact written *for posterity*, while deployments are its ephemeral instantiations.

## How this repository works

This repository builds as an **instructional walkthrough**: a [Zensical](https://zensical.org) site covering the full architecture, from network segmentation through the SSP skeleton.  A deployment is an **overlay**: the institution-specific values live in [`overlay/deployment.yml`](overlay/deployment.yml), and the reference build ships with the fictional Northwinds University **Mudroom** deployment filled in as the worked example.  An institutional deployment repository overrides that one file (and adds its private material) and rebuilds the same site — the walkthrough becomes that institution's deployment documentation.

| Layer | Name | Repository | Visibility |
|---|---|---|---|
| Project / reference architecture | Posternity | `atmarx/posternity` | Public |
| Reference deployment (fictional Northwinds University) | **Mudroom** (`mudroom.northwinds.edu`) | `northwinds/mudroom` | Public |
| Institutional deployment (example) | *institution's choice* (e.g., `mgmt-enclave`) | private | Private |

**Contribution discipline:** anything an institutional deployment needs that is not institution-specific is built in `posternity` and consumed downstream — never patched locally.  The moment a private deployment carries a generalizable fix the reference does not, the reference architecture has started lying.

## Building the site

```sh
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
zensical serve
```

`zensical build --strict` is what CI runs: a broken link, a dead anchor, or an overlay key the docs reference but `overlay/deployment.yml` doesn't define fails the build.  Pushes to `main` publish to [atmarx.github.io/posternity](https://atmarx.github.io/posternity/).

## Status

Draft 0.1 — the specification is complete and lives in [`docs/`](docs/); build documentation, scripts, and the Mudroom reference deployment are in progress.

Contributions of generalizable improvements are welcome; institution-specific logic belongs in your deployment repository.

## License

A docs/code split, because the two get used differently.

- **The specification and walkthrough** — everything under `docs/` — is [**CC BY-SA 4.0**](https://creativecommons.org/licenses/by-sa/4.0/).  Full text in [`LICENSE`](LICENSE).  Same terms as Northwinds, where the Mudroom lives: fork it, adapt it, credit the source, and share your adapted version under the same terms.
- **Code and configuration** — scripts, the overlay, the site build, and any code block or config snippet inside the docs — is [**MIT**](LICENSE-CODE).  Paste the pfSense rule or the intake script into your private config repo and get on with your day.

ShareAlike attaches when you *share*.  A private institutional deployment that rebuilds this site for its own operators, or hands its SSP to an assessor, owes nothing back — though the [contribution discipline](#how-this-repository-works) hopes you send the generalizable fixes upstream anyway.
