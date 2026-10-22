# TRU Quantum Transactions

## A first transaction with TRU Core and tru-cli

This guide walks through a small plain-TRU self-transfer: fund a fresh quantum address **A**, then spend it to a fresh quantum address **B** using WOTS+. It documents the separate quantum wallet commands supplied with Core 0.08.5/0.08.6. Mainnet activation is at block **50,000**; check your own node before starting.

The ordinary funding transaction uses the ordinary wallet signature. The **A → B spend** uses WOTS+ and is verified by `OP_CHECKWOTSVERIFY` (PQ01). This does not convert all your wallet's existing funds to quantum protection. Legacy TRU, token, TRUScript, and contract sending remain separate.

This walkthrough is for **Linux Bash**, a running local Core, `tru-cli`, and Python 3. Run commands one step at a time. Do not paste the whole guide into a terminal.

## Before you start

- Use your own synchronized Core and its local wallet. Verify agreement with a trusted peer at the same block height if the node has recently recovered from a fork.
- Wallet operations require local loopback RPC, an unlocked wallet when applicable, and `TRU_RPC_WALLET_SEND_ENABLE=1` in the **Core process's environment**. Setting it only in a CLI terminal does not change an already-running Core. This is a general wallet-send permission, not a quantum-only switch.
- Start with approximately **0.01 TRU** in one confirmed, ordinary wallet-owned output. Do not select a token, contract, reserved, or immature mining output.
- Each quantum address/descriptor is for **one payment and one spending intent**. Never send funds back to an address already used for signing.
- Back up your wallet and its **current encrypted `.pq01` sidecar together**. The seed alone is insufficient. Stop wallet activity and cleanly stop Core for a consistent file backup; preserve your deployment's actual wallet files and the sidecar path returned by `quantum receive`. Restart the same Core afterward.
- Refresh that backup after generating descriptors and after signing. Never run cloned signing wallets or restore stale key-use state. A chain reorganization does not make a used WOTS key safe to reuse.

### Amounts and fees

**Quantum fund/send/exit spend the entire selected output minus the fee. There is no amount field or change output in this workflow.** The default RPC cap checks the selected input and is **1 TRU** unless the operator changes it.

| Item | Atoms | TRU |
| --- | ---: | ---: |
| One TRU | 100,000,000 | 1 |
| Suggested starting output | 1,000,000 | 0.01000000 |
| Example fee per quantum operation | 10,000 | 0.00010000 |
| A after funding | 990,000 | 0.00990000 |
| B after the WOTS+ send | 980,000 | 0.00980000 |

These amounts also match the completed mainnet self-transfer documented below. Preparing the initial ordinary output has its own fee. The reviewed implementation requires at least 10,000 fee atoms and also checks wallet policy; a fee may need to be higher. Decide the fee **before signing**. Never create a differently signed WOTS transaction to bump its fee.

## 1. Connect to Core and make a private work folder

Set the executable path for your installation. The usual repository directory is `TRU-Core`; some installations use `NEW_TRU`.

```bash
TRUCLI="$HOME/TRU-Core/build-native/bin/tru-cli"
# If your installation uses NEW_TRU, use this instead:
# TRUCLI="$HOME/NEW_TRU/build-native/bin/tru-cli"

test -x "$TRUCLI" && printf 'CLI found\n'
umask 077
QWORK="$(mktemp -d "$HOME/tru-quantum-session-XXXXXX")"
cd "$QWORK"
printf 'Keep this session folder: %s\n' "$QWORK"

"$TRUCLI" raw getconsensus09info '{}'
"$TRUCLI" quantum list
```

Stop if the CLI path is wrong or an RPC call fails. Confirm `active: true` and the expected Core version. Keep this terminal open so its variables remain set. If the sidecar has never been created, `quantum list` may report it missing; generate the first descriptor in step 3. If you previously had a quantum wallet, investigate a missing sidecar instead of silently starting over.

## 2. Select a small ordinary output

```bash
"$TRUCLI" listaddresses
```

Choose one of **your own** ordinary addresses and enter it below:

```bash
read -rp 'Your ordinary wallet address: ' ORDINARY_ADDRESS
"$TRUCLI" listunspent "$ORDINARY_ADDRESS"
```

An output is identified by its transaction ID (`txid`) and output number (`vout`). Use an output with confirmations and `spendable: true`, then verify it with `quantum input` below.

**If you only have large mining rewards:** use your normal wallet Send function to send **0.01 TRU to an ordinary address you own**, then wait for confirmation. The ordinary send creates change; the quantum send does not. You may use the existing CLI send function if it is enabled:

```bash
"$TRUCLI" sendtoaddress "$ORDINARY_ADDRESS" 0.01 --dry-run
```

**The dry run checks syntax, the spending cap, and unlocked-wallet access only. It does not simulate UTXO selection or fees.** Review its result and stop on errors. If the destination and amount are correct, send once:

```bash
"$TRUCLI" sendtoaddress "$ORDINARY_ADDRESS" 0.01
```

Wait for confirmation, then query Core with `listunspent` again. If you saved its output to a JSON file, refresh that file before filtering it; rereading a saved file does not fetch new chain state. Select the **0.01-TRU output**, not the larger change output. If submission times out, inspect transaction history before sending again.

Enter the selected outpoint. `vout` is not necessarily zero:

```bash
read -rp 'Ordinary funding transaction ID: ' FUNDING_TXID
read -rp 'Its output number (vout): ' FUNDING_VOUT

"$TRUCLI" quantum input \
  "{\"txid\":\"$FUNDING_TXID\",\"vout\":$FUNDING_VOUT}"
```

Confirm the amount is what you intend to spend. For this example, `amount_atoms` should be **1000000**. Stop on any error. A one-atom output cannot pay the fee; a 50-TRU reward is too large for this example and exceeds the default quantum input cap.

## 3. Create quantum address A

```bash
"$TRUCLI" quantum receive > quantum-A.json
cat quantum-A.json
```

The response contains `address`, `scriptPubKey`, and `sidecar`. The full address begins `truq1:`. It encodes the receiving script; it is not an ordinary TRU address or a QR image.

Back up the wallet and updated sidecar consistently **before funding A**. Only fund this descriptor once. The `unused` label tracks signing intent; it is not a balance or proof that no payment has arrived.

The following small helper reads saved CLI JSON and rejects errors. It only reads local files; it does not sign or broadcast anything:

```bash
cat > read-quantum-json.py <<'PY'
import json
import sys

try:
    with open(sys.argv[1], encoding="utf-8") as f:
        data = json.load(f)
    if not isinstance(data, dict) or data.get("error"):
        raise ValueError("RPC error or unexpected response")
    data = data.get("result", data)
    if not isinstance(data, dict) or data.get("error"):
        raise ValueError("RPC error or unexpected result")
    field = sys.argv[2]
    value = data.get(field)
    if not isinstance(value, str) or not value:
        raise ValueError("Missing field: " + field)
    if field == "address" and not value.startswith("truq1:"):
        raise ValueError("Not a quantum descriptor")
    if field == "rawHex":
        if data.get("signed") is not True:
            raise ValueError("No completed signature")
        if len(value) % 2 or any(c not in "0123456789abcdefABCDEF" for c in value):
            raise ValueError("Invalid transaction hex")
    print(value)
except (OSError, ValueError, IndexError) as error:
    sys.exit(str(error))
PY

QA="$(python3 read-quantum-json.py quantum-A.json address)" && printf '%s\n' "$QA"
```

If extraction fails, stop. Do not continue with an empty variable.

## 4. Sign the ordinary-to-quantum funding transaction

```bash
"$TRUCLI" quantum fund \
  "{\"txid\":\"$FUNDING_TXID\",\"vout\":$FUNDING_VOUT,\"fee_atoms\":10000,\"destination\":\"$QA\"}" \
  > quantum-fund-signed.json

cat quantum-fund-signed.json
```

Expect `signed: true`, `broadcast: false`, `input_atoms`, `send_atoms`, `fee_atoms`, and `rawHex`. Nothing has been broadcast by this signing command. With a 1,000,000-atom input and 10,000-atom fee, A receives 990,000 atoms.

Extract and decode:

```bash
FUND_HEX="$(python3 read-quantum-json.py quantum-fund-signed.json rawHex)" &&
  "$TRUCLI" decoderawtransaction "$FUND_HEX"
```

Check the single output's script against `scriptPubKey` in `quantum-A.json`. Check the output amount and fee. Stop if anything differs from your intention.

## 5. Broadcast funding and wait for confirmation

Only after reviewing the decoded transaction:

```bash
"$TRUCLI" quantum submit "{\"txHex\":\"${FUND_HEX:?Missing funding hex}\"}" \
  > quantum-fund-submitted.json
cat quantum-fund-submitted.json
```

Save the returned transaction ID. A returned ID or mempool acceptance is **not a block confirmation**.

```bash
read -rp 'Quantum funding transaction ID returned above: ' A_TXID
"$TRUCLI" gettransaction "$A_TXID"
"$TRUCLI" quantum input "{\"txid\":\"$A_TXID\",\"vout\":0}"
```

This workflow creates one output, at index `0`. Wait until the transaction is confirmed on the active chain and `quantum input` recognizes the output as available. Compare its script with A and its amount with your signed funding result. More confirmations provide more protection against a reorganization; one confirmation is not finality.

## 6. Create quantum destination B

For this first self-transfer, generate B in the same wallet:

```bash
"$TRUCLI" quantum receive > quantum-B.json
cat quantum-B.json
QB="$(python3 read-quantum-json.py quantum-B.json address)" && printf '%s\n' "$QB"
```

Confirm B differs from A. Back up the updated wallet and sidecar consistently again.

For a payment to someone else, **the recipient generates B in their own wallet** and sends you the complete fresh `truq1:` descriptor. They retain their own sidecar and keys. Verify the descriptor with them and obtain assurance that it has not already been funded. Do not generate a recipient's private receiving key in your wallet.

## 7. Sign the WOTS+ transaction from A to B

```bash
"$TRUCLI" quantum send \
  "{\"txid\":\"$A_TXID\",\"vout\":0,\"fee_atoms\":10000,\"source\":\"$QA\",\"destination\":\"$QB\"}" \
  > quantum-send-signed.json

cat quantum-send-signed.json
```

**This is the WOTS+ signing step. Treat A as reserved for this exact transaction from this point onward—even if a timeout occurs.** Do not change the destination, fee, or input and sign again with A.

Expect `signed: true` and `broadcast: false`. In the worked amount example, B receives 980,000 atoms. Preserve the signed JSON and a consistent backup of the latest wallet/sidecar state after signing.

```bash
SEND_HEX="$(python3 read-quantum-json.py quantum-send-signed.json rawHex)" &&
  "$TRUCLI" decoderawtransaction "$SEND_HEX"
```

Verify B's script and the amount. If the transaction is wrong, stop and seek assistance; do not create a replacement WOTS signature with A.

## 8. Broadcast and verify receipt

After reviewing the transaction:

```bash
"$TRUCLI" quantum submit "{\"txHex\":\"${SEND_HEX:?Missing signed hex}\"}" \
  > quantum-send-submitted.json
cat quantum-send-submitted.json
```

Record the returned transaction ID:

```bash
read -rp 'Quantum send transaction ID returned above: ' B_TXID
"$TRUCLI" gettransaction "$B_TXID"
"$TRUCLI" quantum input "{\"txid\":\"$B_TXID\",\"vout\":0}"
"$TRUCLI" quantum list
```

The self-test is complete when the send is confirmed on the active chain and B's output is confirmed, unspent, and has the expected script and amount. A should no longer be treated as a receiving address. B may still appear `unused` because it has not signed a spend—that does not mean it has no funds.

To spend B later, create a new destination C and use B as the source. Never fund A again, and never send change back to A.

## Recovery: timeout, disconnect, or uncertain submission

Do not repeatedly click Send or create a new signature. Preserve all files and the latest sidecar.

For a WOTS send whose response was lost, retrieve the saved transaction:

```bash
"$TRUCLI" quantum retry "{\"source\":\"${QA:?Missing source descriptor}\"}" \
  > quantum-recovered.json

RECOVERED_HEX="$(python3 read-quantum-json.py quantum-recovered.json rawHex)" &&
  "$TRUCLI" decoderawtransaction "$RECOVERED_HEX"
```

`retry` retrieves saved bytes; it does not broadcast. Use the transaction ID saved from submission to check `gettransaction`, and inspect the mempool if needed. In the observed 0.08.6 output, `decoderawtransaction` returned an empty top-level `txid`; do not treat that field as a usable ID. If no submission ID was captured, retain the exact saved bytes and investigate status before rebroadcasting; never create a different WOTS signature to recover an ID. If it is already confirmed, do not submit again. If rebroadcast is needed, submit **only those exact saved bytes**:

```bash
"$TRUCLI" quantum submit "{\"txHex\":\"${RECOVERED_HEX:?Missing recovered hex}\"}"
```

If recovery says no completed transaction is saved, or the descriptor is reserved without saved bytes, stop and seek assistance. Do not reset its state or restore an older sidecar. `quantum retry` is for a quantum source's saved spend, not the ordinary funding operation; preserve `quantum-fund-signed.json` for funding recovery.

## Optional: exit to an ordinary TRU address

`quantum exit` spends a quantum output to an ordinary address. It still consumes the source's WOTS signing intent, but the resulting ordinary output is **no longer protected by a WOTS spending condition**.

Use a confirmed quantum output whose descriptor has never signed before. Its JSON fields are:

```json
{
  "txid": "CONFIRMED_QUANTUM_TXID",
  "vout": 0,
  "fee_atoms": 10000,
  "source": "UNUSED_FOR_SIGNING_QUANTUM_DESCRIPTOR",
  "destination": "YOUR_ORDINARY_TRU_ADDRESS",
  "confirm_exit": true
}
```

Pass that JSON to `tru-cli quantum exit`, save its response, decode and review its `rawHex`, then explicitly submit as above. Placeholder text is explanatory, not a usable address or transaction ID.

## Command reference

| CLI command | Purpose | Broadcasts? |
| --- | --- | --- |
| `quantum receive` | Creates a fresh descriptor and saves its wallet state | No |
| `quantum list` | Lists descriptors and signing-state indicators; not balances | No |
| `quantum input '{"txid":"...","vout":0}'` | Checks a confirmed, available plain-TRU input | No |
| `quantum fund '{...}'` | Signs ordinary TRU → quantum output | No |
| `quantum send '{...}'` | Signs quantum → fresh quantum output with WOTS+ | No |
| `quantum exit '{...}'` | Signs quantum → ordinary output with explicit acknowledgement | No |
| `quantum retry '{"source":"..."}'` | Retrieves saved bytes for the source descriptor | No |
| `quantum submit '{"txHex":"..."}'` | Submits the signed transaction | **Yes** |

RPC mapping: `receive/list/fund/send/exit/retry` use `quantumwallet` with an `action` field; `input` uses `getquantuminput`; `submit` uses `sendquantumtransaction`. For example:

```bash
"$TRUCLI" raw quantumwallet '{"action":"list"}'
"$TRUCLI" raw getconsensus09info '{}'
```

## Confirmed mainnet example — October 21, 2026

The project operator completed this self-transfer on **gw878 running TRU Core 0.08.6**, after mainnet PQ01 activation at height **50,000**. The supplied RPC results show funding, WOTS+ signing, submission, block inclusion, and a confirmed unspent destination output. The confirmation counts below are those observed in the transcript, not live counters.

| Stage | Block | Amount in destination | Observed result |
| --- | ---: | ---: | --- |
| Ordinary self-payment to prepare the input | 52,685 | 0.01000000 TRU | Confirmed; selected output index 0 |
| Ordinary-signature funding of quantum A | 52,690 | 0.00990000 TRU | Confirmed; 3 confirmations in the supplied check |
| WOTS+ spend from quantum A to quantum B | **52,696** | **0.00980000 TRU** | **Confirmed; 1 confirmation in the supplied check** |

Each of the funding and WOTS+ spend transactions paid **0.00010000 TRU**. Together those two fees total **0.00020000 TRU**, excluding the earlier ordinary self-payment fee.

### Public transaction references

**Ordinary preparation transaction:**

```text
e53e542ca52f1083a1caea93602e9c1c14d4038fcc195578b7464f3a80bd7ce6
```

**Quantum funding transaction (A, output 0):**

```text
75450982cb6a094e5ccbfc977bb154908576c0f04ccc32a6124f0ea5188a9f78
```

**WOTS+ A → B transaction (B, output 0):**

```text
14ec11a1a75c4a9eb62e451a7d63489c85bba2ddaff06274523f1e59858d71ac
```

**Block 52,696 hash:**

```text
9b59418450e0e880eea7a25872239a2de32154ac6e85595a789c650100000000
```

The WOTS+ transaction returned `accepted: true` on submission. A subsequent `gettransaction` returned `active: true`, `txState: "CONFIRMED"`, `blockheight: 52696`, and `reorgSignal: false`. `quantum input` then returned **980000 atoms** at output 0, with B's expected script:

```text
512078bdf82aa24d22d83b8d1503ff1784ea4c39c96b4e10b0e7e7c8387912625a94200680696ffe2b4f598b7cd37b1f4c82e36c8e903a6b99650f68ded0b25e29b758bb51
```

### Verify the historical example on your own synchronized node

These calls are read-only:

```bash
"$TRUCLI" gettransaction 14ec11a1a75c4a9eb62e451a7d63489c85bba2ddaff06274523f1e59858d71ac
"$TRUCLI" getblockbyheight 52696
```

Compare the transaction's block reference with the block hash above. Current confirmations can differ, and reorganization can change active-chain inclusion. The successful `quantum input` result recorded here proves the output was available at that observation; if B is later spent, that input check will no longer succeed.

**These transaction IDs and scripts are historical examples, not inputs or receiving addresses for a new tutorial run. Generate your own descriptors and select your own small output. Do not send funds to the example descriptors.**

### Wallet state after the transfer

| Descriptor | Reported state | Meaning |
| --- | --- | --- |
| A | `reserved_or_signed`; `has_saved_transaction: true` | A has signed its spending transaction. Preserve its state; never fund it again or sign a different transaction with it. |
| B | `unused`; `has_saved_transaction: false` | B has received its payment but has not signed a spend. It is funded, not empty. Do not fund it again; spend to a fresh C when needed. |

This demonstrates the **fund → WOTS+ sign → submit → confirm → receive** path on the operator's active mainnet chain. It does not establish independent verification by every peer or constitute an independent cryptographic audit. Keep current wallet and sidecar backups together; never publish seeds, private keys, wallet files, or sidecars.

## Troubleshooting

| Message or symptom | What to do |
| --- | --- |
| Could not reach node | Check the Core process, RPC listener, and CLI configuration. A running P2P service does not establish that RPC is available. |
| `active: false` in `getconsensus09info` | Wait for activation/synchronization; do not bypass the activation check. |
| `active: false`, `txState: MEMPOOL`, zero confirmations in `gettransaction` | The transaction is not yet in the active chain. Wait for mining; this does not mean PQ01 activation is disabled. |
| Empty `txid` in `decoderawtransaction` | Seen in the tested build. Use the ID returned by submission for confirmation checks; do not invent or guess an ID. |
| Requires loopback / wallet-send enable | Use the local node and its operator-approved startup environment. Do not expose wallet signing through a public RPC service. |
| Invalid address / invalid character | Replace placeholders with your own actual address; copy the entire descriptor without extra characters. |
| Selected input exceeds cap | Prepare a small ordinary output; do not raise the cap just to use a mining reward for this tutorial. |
| Input must be confirmed and unspent | Check transaction confirmation, output index, and whether it was already spent or reserved. |
| Fee below policy | Stop. Determine an acceptable fee before any WOTS signing attempt. If signing may already have happened, recover saved bytes first. |
| Sidecar missing / authentication failed | Verify the correct wallet and matching current sidecar. Do not delete or reset state to bypass the error. |
| Empty or malformed saved JSON | The command may have failed or timed out. Do not proceed with an empty hex variable or overwrite prior signing evidence. |

## Documentation basis

Prepared for TRU Core 0.08.6 from the operator's supplied command transcript and the available quantum wallet implementation (`quantum_core_wallet_v1.inc`, `quantum_wallet_v1.h`, `quantum_chain_v1.h`, `rpc_server.cpp`, and `tru-cli.cpp`). Updated with the operator-supplied mainnet transaction receipts for blocks 52,685, 52,690, and 52,696. The operator executed the transactions; preparation of this document performed no signing or broadcasting. This is an operational guide and recorded functional example, not an independent cryptographic audit.
