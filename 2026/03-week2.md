# Week 2 — March 2026

> Running your own nodes: Bitcoin pruned node + Ethereum light client on a €150 mini PC.

---

## Why Run Your Own Node?

Most crypto infrastructure depends on third-party RPC providers — Infura, Alchemy, Ankr. They work, but you're trusting them blindly. They can:
- Go down (and take your bots with them)
- Return incorrect data
- Log your queries (privacy)

Running your own node means you validate independently. For Bitcoin, this is full sovereignty. For Ethereum, a light client at minimum means you *verify* what the RPC tells you instead of blindly accepting it.

---

## Bitcoin Pruned Node on M900 Tiny

**Hardware:** Lenovo ThinkCentre M900 Tiny (i5-6500T, 8GB RAM, 477GB SSD)

A Bitcoin full node requires ~600GB. The M900 doesn't have headroom for that. Solution: pruned node.

**What pruned means:** Bitcoin Core downloads and validates the full blockchain during sync, but only keeps the most recent blocks on disk afterward. You still fully validate every transaction — you just don't store history.

```bash
# /home/m900/.bitcoin/bitcoin.conf
prune=2000      # keep last 2GB of blocks
server=1        # enable RPC
listen=0        # no inbound connections (safer for hot wallet)
dbcache=512     # limit RAM usage
```

**Install:**
```bash
# Download Bitcoin Core 27.2 from bitcoincore.org (verify SHA256)
wget https://bitcoincore.org/bin/bitcoin-core-27.2/bitcoin-27.2-x86_64-linux-gnu.tar.gz
tar -xzf bitcoin-27.2-x86_64-linux-gnu.tar.gz
sudo install -m 0755 bitcoin-27.2/bin/bitcoind /usr/local/bin/bitcoind
sudo install -m 0755 bitcoin-27.2/bin/bitcoin-cli /usr/local/bin/bitcoin-cli
```

**Run as systemd service:**
```ini
[Unit]
Description=Bitcoin Core pruned node
After=network.target

[Service]
User=m900
ExecStart=/usr/local/bin/bitcoind -conf=/home/m900/.bitcoin/bitcoin.conf -datadir=/home/m900/.bitcoin
ExecStop=/usr/local/bin/bitcoin-cli -conf=/home/m900/.bitcoin/bitcoin.conf stop
Restart=on-failure
RestartSec=30

[Install]
WantedBy=multi-user.target
```

**Disk usage:** ~10GB total (vs ~600GB for a full node).  
**RAM:** ~600MB during operation.  
**Initial sync:** several hours — downloads all headers, then recent blocks only.

**Note on Bitcoin Core 27.2 and wallets:** The old BDB/legacy wallet format is deprecated. Use descriptor wallets. `dumpprivkey` no longer works — use `listdescriptors true` to export private key material.

---

## Ethereum Light Client — Helios

For Ethereum, a full node is ~500GB. A pruned node still requires ~250GB (Erigon). Neither fits comfortably on the M900 alongside everything else.

**Helios** is a light client by a16z. It proxies an existing RPC endpoint (Infura, Alchemy, Ankr) but *verifies* every response against Ethereum consensus headers. Zero disk usage. You don't need to trust the upstream provider.

```
Your app → localhost:8545 (Helios) → verifies → Infura/Alchemy
```

### The install problem (not documented anywhere)

The official install method fails:
```bash
cargo install helios  # WRONG — installs a completely different package
cargo install --git https://github.com/a16z/helios helios  # WRONG — v0.11 is now a library, no binary
```

**What actually works** (Helios v0.11):
```bash
git clone --depth 1 https://github.com/a16z/helios
cd helios
cargo build --release --package helios-cli  # CLI is a separate crate
sudo cp target/release/helios /usr/local/bin/helios
```

Helios v0.11 restructured the repo — the CLI binary moved to the `cli/` subdirectory as a separate package (`helios-cli`). The top-level crate is now a library only.

### Running Helios

```bash
# Helios v0.11 uses subcommands
helios ethereum \
  --execution-rpc https://mainnet.infura.io/v3/YOUR_KEY \
  --rpc-bind-ip 127.0.0.1 \
  --rpc-port 8545
```

**Verify it works:**
```bash
curl -s http://localhost:8545 -X POST \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}'
# → {"jsonrpc":"2.0","result":"0x177a562","id":1}
```

Run as a systemd service (same pattern as bitcoind above).

---

## What's Next

- Bitcoin node finishes syncing → generate wallet, monitor balance
- Ethereum L2 nodes (Arbitrum/Base) — viable on a VPS when that's set up
- Evaluate Erigon for a proper Ethereum L1 node on future hardware
