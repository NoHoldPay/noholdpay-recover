# NoHoldPay self-recovery CLI

Signed binaries for `noholdpay-recover`, the offline tool that lets a NoHoldPay
merchant reach their own funds without NoHoldPay.

NoHoldPay is non-custodial: merchants supply their own wallets and the platform
never holds customer funds. This tool re-derives every deposit address the
platform generated for you and, with your own key, sweeps them. It talks only to
a public blockchain RPC. It never contacts NoHoldPay, and it works whether or
not NoHoldPay still exists.

## What you need

- Your **recovery kit** (`recovery-kit.json`), downloaded from Settings in the
  merchant dashboard. Download a fresh one whenever you add or change a wallet.
- A **public RPC URL** for the chain you are recovering.
- **Your own key.** The platform cannot move your funds and neither can this
  tool without you.

Keep the kit somewhere you can reach when the platform is down. It is not
secret in the sense of a private key, but it is what makes recovery quick.

## Which file do I download?

| Your computer | Download this file |
| --- | --- |
| **Mac** with Apple Silicon (M1, M2, M3, M4) | `noholdpay-recover-v1.0.0-darwin-arm64` |
| **Mac** with an Intel processor | `noholdpay-recover-v1.0.0-darwin-amd64` |
| **Windows** | `noholdpay-recover-v1.0.0-windows-amd64.exe` |
| **Linux** on a normal PC or server | `noholdpay-recover-v1.0.0-linux-amd64` |
| **Linux** on ARM (Raspberry Pi, ARM server) | `noholdpay-recover-v1.0.0-linux-arm64` |

"darwin" is the internal name for macOS and "amd64" means a normal
64-bit Intel or AMD processor, not an AMD-only one. The file names use
those terms because that is what the checksums and signatures are issued
against, so they have to match exactly.

**Not sure which Mac you have?** Apple menu, then About This Mac. If the
Chip line says Apple M1, M2, M3 or M4, take Apple Silicon. If it says
Intel, take the Intel file.

**Not sure on Windows?** Almost every Windows PC takes the `windows` file
above. There is only one.

Download each file with the matching `.sigstore.json` beside it - that is
the signature, and you need both to verify.

## Verify before you run it

This binary will handle a key that controls your funds. Verify it first.

**1. Checksum.** Compare against the value in your recovery kit, or the
`SHA256SUMS` file attached to the release:

```
sha256sum noholdpay-recover-v1.0.0-linux-amd64
```

**2. Signature.** Every binary is signed with Sigstore Cosign using GitHub's
OIDC identity, and the signature is recorded in a public transparency log. The
`.sigstore.json` bundle beside each binary carries the signature, certificate,
and log proof:

```
cosign verify-blob \
  --bundle noholdpay-recover-v1.0.0-linux-amd64.sigstore.json \
  --certificate-identity-regexp '^https://github\.com/NoHoldPay/noholdpay/\.github/workflows/release-recover-cli\.yml@refs/tags/recover-cli/' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  noholdpay-recover-v1.0.0-linux-amd64
```

This proves the binary was produced by that exact workflow, from a specific
commit, and has not been altered since. Verification is entirely offline
against public logs.

The capitalisation in that identity is significant. It is matched as a
case-sensitive regular expression against the signing certificate, so
`NoHoldPay` must be spelled exactly as shown.

## Run it

Scanning is read-only. It finds and prints funded addresses and moves nothing:

```
noholdpay-recover --kit recovery-kit.json --rpc <public-rpc-url>
```

To build recovery transactions, add `--sweep`. On its own that still moves
nothing - it prints the transaction data. Broadcasting requires you to supply a
key explicitly:

```
noholdpay-recover --kit recovery-kit.json --rpc <public-rpc-url> --sweep \
  --signer-key-file /path/to/key
```

Solana uses `--solana-control-key-file` instead, since a Solana forwarder is
swept with the control key you registered.

There is no way to pass a private key as a command-line argument. The old
`--signer-key` flag is permanently disabled, because arguments end up in shell
history and process listings. Keys come from a file or from stdin, and the tool
asks for confirmation before broadcasting anything.

Run `noholdpay-recover --help` for the full flag list, and
`noholdpay-recover --version` to confirm the build matches your kit.

## Why there is no source code here

The NoHoldPay platform is proprietary, so this repository publishes compiled
binaries only. That is a deliberate choice, and it does not require you to
trust the binary.

**On EVM chains and TRON you can check its work directly.** The forwarder
factory exposes address computation as a public, read-only contract function.
Take any address this tool prints, ask the factory to compute the same address
from a public RPC, and compare. The blockchain is the authority, not the tool.
If they disagree, do not proceed.

**The signature proves provenance.** It binds each binary to a specific build,
recorded in a public log that neither we nor anyone else can alter after the
fact.

The binaries are licensed under Apache-2.0 (see `LICENSE.txt` in each release),
so you are free to copy, mirror, and redistribute them. That is intentional: a
recovery tool you needed our permission to share would be useless in the exact
situation it exists for.

## What this tool cannot do

- It cannot move funds without your key. No key, no transaction.
- Solana forwarder balances require the control key you registered. If that key
  is lost and NoHoldPay is unreachable, those balances cannot be moved, because
  re-pointing a control key needs a platform-signed instruction.
- Solana token accounts whose mint carries a token extension are reported but
  skipped. The on-chain program rejects those mints rather than guess how a
  transfer fee or hook should affect a payment, and this tool mirrors that
  exactly rather than building a transaction that would fail.

## Documentation

Full recovery walkthrough, including how to read your recovery kit:
https://docs.noholdpay.com/docs/wallets/recovery-kit

If NoHoldPay is reachable, the dashboard's "Sweep manually" button does the same
job without any of this.
