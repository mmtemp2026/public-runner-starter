# Cloudflare WARP on the runners

## What it does

Installs the `cloudflare-warp` client and connects it, so everything the runner
sends afterwards leaves through Cloudflare's consumer WARP pool instead of
GitHub's Azure ranges:

```
IP before WARP: 132.196.36.80    <- Azure datacenter
IP after WARP:  104.28.216.161   <- Cloudflare consumer
```

Two reasons that helps with BookMyShow. The exit looks like consumer traffic
rather than a datacenter, and BMS itself sits behind Cloudflare, so the request
arrives intra-network. `bms9.py` already names the underlying problem in a
comment: *"datacenter egress IPs are blocked on reputation, not rate."*

## Credentials

**None.** The free consumer tier registers anonymously — `warp-cli registration
new` mints an account on the runner at job time and throws it away with the VM.
No secret, no login, no account to manage, nothing to rotate.

Credentials would only enter the picture for WARP+ / Zero Trust (a device token
or an enrolled team). Not needed here, and not worth it unless the free pool
turns out to be rate-limited.

## What it does NOT do

**WARP does not raise the per-IP rate ceiling.** It changes *which* IP you are,
not how much that IP may ask for.

WARP swaps a flagged IP for a fresher one. It does not stop you flagging the
fresh one. The rate win has to come from somewhere else.

## Where the rate win actually comes from

The matrix split, not WARP.

These workflows used to run `railway_runner.py` with `SHARD_IDS=1-8`, so all
eight Chromium shards shared one runner:

```python
with ThreadPoolExecutor(max_workers=len(shards))   # railway_runner.py:113
```

One job is one runner is one exit IP — roughly 15 req/s at BookMyShow from a
single address, exactly what `local_runner.py`'s docstring warns about:

> Concurrency defaults to 2, **NOT 8**. Eight shards at once is ~15 req/s from a
> single IP, which is what gets egress IPs blocked.

All four workflows now run **one shard per matrix job**, so eight runners mean
eight IPs and eight WARP registrations. Per-IP rate drops ~8x before WARP is
applied at all; WARP then handles reputation on top. That ordering matters when
you are debugging: if blocks persist, suspect the split before the VPN.

MintBox04/Workflows reaches the same place with 15 jobs for 15 shards.

They exclude their District shard from WARP (`if: matrix.shard != 15`). Ours is
excluded structurally — District runs on Railway, not on a GitHub runner, so
nothing here touches its egress.

## Operating it

- Turn it off with repo variable `WARP_ENABLED=false`. No edit, no revert.
- The step **fails the job** if the egress IP did not change. MintBox's version
  prints the IP but never checks it, so a silently failed WARP falls back to the
  Azure address and looks identical to a healthy run. Assert, do not assume.
- Costs roughly 40 s of setup per job.
- WARP exit IPs are shared globally across all WARP users, so their reputation
  is not guaranteed and could itself attract limits. If blocks persist *after*
  the matrix split, WARP is the first thing to test by toggling off.
- Free WARP carries no SLA. `WARP_ENABLED=false` is the escape hatch if the pool
  degrades.
