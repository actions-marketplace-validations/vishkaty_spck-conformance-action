# UCP Conformance Action (spck)

Add a **behavioral** [Universal Commerce Protocol](https://ucp.dev) conformance
gate to your CI: this action runs the [spck-conformance](https://pypi.org/project/spck-conformance/)
suite against your UCP merchant server and **fails the build on MUST
deviations** — it executes checkout flows, error contracts, and transport
behavior against the spec, rather than only linting your `/.well-known/ucp`
manifest.

Every check cites the spec requirement it enforces, and the suite itself is
validated both ways before release: each check must clean-pass a known-good
reference server *and* catch every defect injected for it (a check that can't
prove both is reported inconclusive, never green).

> Unofficial. Not affiliated with or endorsed by the UCP project. A pass
> reflects only the checks run against your server, not certified compliance.

## Usage

### Gate a merchant server

```yaml
jobs:
  ucp-conformance:
    runs-on: ubuntu-latest
    steps:
      - name: Start your UCP server
        run: ./scripts/start-server.sh   # serve http://localhost:8080

      - name: UCP conformance gate
        uses: vishkaty/spck-conformance-action@v1
        with:
          server-url: 'http://localhost:8080'
          config: '.github/ucp-merchant-config.json'   # optional, unlocks lifecycle checks

      - name: Upload conformance report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: spck-report
          path: spck-report.xml
```

The JUnit report renders in any CI test viewer; entries are named by their
register check ids, and deviations carry the spec citation for the requirement
they violated.

### Run the agent-side lane (no server needed)

If you build a shopping **agent**, the reverse harness runs a reference agent
against an adversarial sandbox — self-contained:

```yaml
      - uses: vishkaty/spck-conformance-action@v1
        with:
          agent-lane: 'true'
```

## Inputs

| Input | Default | Description |
|---|---|---|
| `server-url` | — | Base URL of the UCP merchant server to test. Exactly one of `server-url` / `agent-lane`. |
| `agent-lane` | `false` | `'true'` runs the self-contained agent-side lane instead. |
| `config` | — | Merchant config JSON path (product ids, discount codes, …) to unlock the full lifecycle checks. Scaffold with `spck-conformance --server URL --init`. |
| `junit-path` | `spck-report.xml` | Where the JUnit XML report is written (server mode). |
| `package-version` | latest | Pin a specific `spck-conformance` release. |
| `python-version` | `3.12` | Python used to run the suite. |

## Exit behavior

| Result | Build |
|---|---|
| All applicable MUSTs pass | ✅ success |
| Any MUST deviation (exit 2) | ❌ failure, deviations listed with spec citations |
| Discovery/transport failure — not a UCP endpoint (exit 1) | ❌ failure |
| Both or neither mode selected (exit 64) | ❌ configuration error |

## What it checks

The merchant lane covers discovery, checkout lifecycle, cart, orders,
fulfillment, discounts, payment handlers, error envelopes, idempotency, and
transport behavior across UCP versions `2026-01-11`, `2026-01-23`, and
`2026-04-08` — the version is auto-detected from your server's discovery
profile. Details, the full check register, and a hosted version of the same
suite: [spck.dev](https://spck.dev).

This action's own CI proves the gate both ways on every change and weekly
against the latest release: the agent lane must pass, a non-UCP server must
fail the build, and a full run against the UCP reference sample merchant must
produce a JUnit report consistent with the gate's verdict.

## License

MIT
