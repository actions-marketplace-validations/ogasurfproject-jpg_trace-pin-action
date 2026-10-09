# TRACE record time anchor

A GitHub Action that pins a signed [TRACE](https://github.com/agentrust-io/trace-spec) Trust Record to a NENRIN ledger, so the record can still be checked after TRACE's 24 hour freshness window.

A TRACE record's only time is `iat`, which the issuer writes, and a conformant verifier refuses a record older than 24 hours. TRACE's informative page [Verifying a record after the freshness window](https://github.com/agentrust-io/trace-spec/blob/main/docs/verifying-after-the-freshness-window.md) describes the later check: replay the freshness comparison at a time T taken from evidence someone other than the issuer holds. A pin gives one such T. The ledger keeps the exact bytes and stamps their sha256 into Bitcoin in a daily batch.

## Use

```yaml
- uses: ogasurfproject-jpg/trace-pin-action@v1
  id: pin
  with:
    record: trace.json
    publish: "true"
- run: echo "pinned ${{ steps.pin.outputs.sha }}"
```

Without `publish: "true"` the action is a dry run: it checks the record and sends nothing. That is the default on purpose, because **a pinned record is public and cannot be withdrawn**.

| input | default | |
|---|---|---|
| `record` | (required) | path to the signed Trust Record |
| `publish` | `false` | `true` to pin |
| `ledger` | `https://ledger.horizonshield.dev` | any service with the same contract |
| `receipt` | `trace-pin-receipt.json` | where the receipt is written |

Outputs: `sha`, `stage` (`dry_run`, `pinned`, or the stage that refused), `status` (`pending` or `anchored`), `url`, `exit-code`. When the step fails (a refused record, a mismatched receipt), read the receipt file: it is always written, and it carries `refused` and `why`.

## What it does

It runs `nenrin-trace-pin` from [nenrin-verify](https://www.npmjs.com/package/nenrin-verify) 0.5.1, pinned to that version:

1. The record goes through the same intake the ledger runs (signature over the RFC 8785 form under `cnf.jwk`, profile, required members, `iat` not in the future). A record the ledger would refuse is refused here and never sent.
2. With `publish: "true"` it is sent, and the ledger's answer must name the sha256 computed locally, or the step fails.
3. Later, anyone can check it without this action: `npx -p nenrin-verify nenrin-trace-pin status <sha>` reports pending or anchored, the Bitcoin block, and recomputes the sha256 of the bytes the ledger serves.

## What a pin does not establish

It bounds when the bytes existed. It does not make any claim inside the record true, and it does not show that the key in `cnf.jwk` belongs to the subject; compare the key thumbprint with a key you already trust (the ledger names the producer when the key is listed in [trace-registry](https://github.com/agentrust-io/trace-registry) `producers/`).

## Licence

MIT. Operated by HORIZON SHIELD, The HORIZONs Co., Ltd., which also runs one of the two registered mirrors of the TRACE registry. Not affiliated with or endorsed by the TRACE project or the Linux Foundation.
