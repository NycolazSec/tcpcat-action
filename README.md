# tcpcat-action

Check, on every push or on a schedule, that your network only allows the flows
you declared. The action runs [tcpcat](https://github.com/NycolazSec/tcpcat)'s
segmentation policy check from the runner and sends violations to GitHub code
scanning as SARIF.

```yaml
# .github/workflows/segmentation.yml
name: segmentation
on:
  schedule: [{ cron: "0 6 * * *" }]
  workflow_dispatch:

permissions:
  contents: read
  security-events: write   # to upload the SARIF report

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      - uses: NycolazSec/tcpcat-action@v1
        with:
          policy: segmentation.yaml
          from: internet          # this runner stands for "the internet"
```

```yaml
# segmentation.yaml
version: 1
name: public exposure
zones:
  shop: [shop.example.com]
rules:
  - name: only HTTPS is public
    from: internet
    to: shop
    ports: "1-1024"
    allow: [443/tcp, 80/tcp]   # anything else reachable is a violation
    require: [443]             # must stay reachable
```

The step fails when a port is reachable that the policy does not allow, or
when a required port is not reachable. Each violation appears in the
repository's **Security → Code scanning** tab.

Only scan hosts you are authorized to test.

## Inputs

| Input | Default | |
|---|---|---|
| `policy` | | Path to a tcpcat segmentation policy (YAML). See [the format](https://github.com/NycolazSec/tcpcat#segmentation-testing-as-code). |
| `from` | `ci` | The vantage point this runner stands for; only the rules whose `from` matches are checked. |
| `args` | | Arguments for a plain scan instead of a policy check, e.g. `-sT -p 22,80,443 example.com`. Ignored when `policy` is set. |
| `version` | `latest` | tcpcat release to use, e.g. `v1.7.0`. |
| `sarif` | `tcpcat.sarif` | Where to write the SARIF report. |
| `upload-sarif` | `true` | Upload the report to code scanning (needs `security-events: write`). |
| `fail-on-violation` | `true` | Fail the step on a violation. |

## Outputs

| Output | |
|---|---|
| `exit-code` | `0` compliant, `1` violation, `2` usage or configuration error. |
| `sarif` | Path of the SARIF report. |

## How it works

- The action downloads the tcpcat release for the runner's architecture
  (Linux x64 or arm64). It checks the archive's SHA-256 against the release's
  `checksums.txt` and refuses to run if they differ.
- It then runs `tcpcat policy check <policy> --from <from> --sarif <sarif>`.
  The policy check uses plain TCP connect probes, so it needs no root and no
  special capabilities.

A GitHub-hosted runner probes from GitHub's network, the "internet" vantage
point. To check internal segmentation, run the same workflow on a
self-hosted runner placed in the zone you want to test.

## License

Apache-2.0 for this action. tcpcat itself is AGPL-3.0 or commercial.
