# Strategy commitments and independent provenance

> **Status:** Optional, independent integration pattern. This is not a Polymarket protocol contract, Builder attribution mechanism, oracle, or leaderboard source.

Use a commit/reveal registry when an agent or builder needs to prove that a strategy description existed before its result was known, then attach outcome evidence later without giving the registry custody or trading authority.

## When this pattern fits

- Pre-register a strategy thesis before submitting Polymarket orders.
- Bind a prediction-market condition ID or Perps instrument ID to that thesis.
- Reveal the strategy after a chosen embargo window.
- Attach immutable outcome and evidence hashes after positions settle.
- Build an independent strategy track record that can be checked on-chain.

Do not use this pattern as a substitute for Polymarket order, trade, position, resolution, or Builder APIs. A registry entry proves what was committed and when; it does not prove that an order was submitted, a fill settled, a market resolved correctly, or volume belongs on an official leaderboard.

## Commitment model

A safe commitment binds the deployment domain and every value that must not be changed later:

```text
domain = keccak256(abi.encode(
  keccak256("StrategyCommitmentRegistry:v2"),
  chainId,
  registryAddress
))

commitmentHash = keccak256(abi.encode(
  domain, author, authorNonce, strategyRefHash, marketRefHash,
  uriContentHash, resolver, salt, revealWindow, resolveWindow
))
```

The domain prevents replay across chains or registry deployments. Author and nonce prevent another wallet from revealing a copied commitment. Both time windows must be part of the preimage so an author cannot change lifecycle semantics after committing.

At commit time, publish only the aggregate hash and the windows. Keep the salt, references, resolver, and URI private until reveal. Use a cryptographically random 32-byte salt; predictable strategy text is vulnerable to dictionary attacks even when hashed.

## Mapping Polymarket data

| Registry field | Suggested Polymarket value |
|---|---|
| `strategyRefHash` | Hash of a canonical strategy document, parameter set, or signed manifest |
| `marketRefHash` | Hash of the Gamma condition ID, CLOB token IDs, or Perps instrument ID plus environment/version |
| `uriContentHash` | `keccak256(bytes(uri))` for an immutable IPFS/Arweave URI or other content-addressed reference |
| `outcomeHash` | Hash of the claimed result: market resolution, realized P&L interval, or terminal position summary |
| `evidenceHash` | Hash of supporting trade IDs, transaction hashes, confirmed fills, and calculation inputs |
| `resolver` | An explicitly selected evaluator; use a separate wallet or governed resolver for independent scoring |

Canonicalize structured documents before hashing. Include a schema version, chain ID, wallet/account scope, market identifiers, observation interval, fee assumptions, and whether P&L is realized or mark-to-market.

## TypeScript lifecycle example

This example uses ethers v6 and a minimal ABI. Keep the preimage in durable private storage until reveal; losing it makes the commitment unrevealable.

```typescript
import { Contract, Wallet, JsonRpcProvider, keccak256, randomBytes, toUtf8Bytes } from "ethers";

const abi = [
  "function authorNonces(address) view returns (uint64)",
  "function hashCommitment(address,uint64,bytes32,bytes32,bytes32,address,bytes32,uint64,uint64) view returns (bytes32)",
  "function commitmentId(address,uint64,bytes32) view returns (bytes32)",
  "function commit(bytes32,uint64,uint64) returns (bytes32)",
  "function reveal(bytes32,bytes32,bytes32,bytes32,bytes32,address,uint64,uint64,string)",
  "function resolve(bytes32,int256,bytes32,bytes32)",
  "function getCommitment(bytes32) view returns ((address author,address resolver,uint64 committedAt,uint64 revealDeadline,uint64 resolutionDeadline,uint64 resolveWindow,uint64 nonce,uint8 status,bytes32 commitmentHash,bytes32 strategyRefHash,bytes32 marketRefHash,bytes32 uriContentHash,bytes32 outcomeHash,bytes32 evidenceHash,int16 scoreBps))"
];

const provider = new JsonRpcProvider(process.env.POLYGON_RPC_URL);
const signer = new Wallet(process.env.PRIVATE_KEY!, provider);
const registry = new Contract(REGISTRY_ADDRESS, abi, signer);

const nonce = await registry.authorNonces(signer.address);
const salt = `0x${Buffer.from(randomBytes(32)).toString("hex")}`;
const strategyRefHash = keccak256(toUtf8Bytes(canonicalStrategyDocument));
const marketRefHash = keccak256(toUtf8Bytes(`polymarket:condition:${conditionId}`));
const uri = "ipfs://...";
const uriContentHash = keccak256(toUtf8Bytes(uri));
const resolver = RESOLVER_ADDRESS;
const revealWindow = 24 * 60 * 60;
const resolveWindow = 7 * 24 * 60 * 60;

const commitmentHash = await registry.hashCommitment(
  signer.address, nonce, strategyRefHash, marketRefHash, uriContentHash,
  resolver, salt, revealWindow, resolveWindow
);
const id = await registry.commitmentId(signer.address, nonce, commitmentHash);

await (await registry.commit(commitmentHash, revealWindow, resolveWindow)).wait();

// Later, no later than revealDeadline:
await (await registry.reveal(
  id, salt, strategyRefHash, marketRefHash, uriContentHash,
  resolver, revealWindow, resolveWindow, uri
)).wait();

// The selected resolver may later record a bounded score and evidence:
await (await registry.connect(resolverSigner).resolve(
  id, scoreBps, outcomeHash, evidenceHash
)).wait();
```

Never send wallet private keys, CLOB credentials, delegated Perps session credentials, or Builder secrets to a registry. Signing and trading remain in the normal Polymarket client flow.

## Lifecycle and resolver rules

- Reveal succeeds through the exact reveal deadline; after it, anyone may mark the record `Expired`.
- Resolution succeeds through the exact resolution deadline; after it, anyone may mark the record `Unresolved`.
- Anchor the resolution deadline to the commit-time reveal deadline, not the reveal transaction time. A late reveal must not extend the resolver window.
- Treat resolver scoring as an attestation, not objective truth. Publish the scoring method and evidence schema before committing.
- Index events for URI text and discovery; do not assume every client will scan contract storage directly.

## Independent Polygon proof of concept

The following deployment demonstrates this pattern on Polygon mainnet. It is independently deployed and is **not an official Polymarket contract or endorsement**. Integrators must review the source, bytecode, trust model, and resolver policy before use.

- Registry: `0x8b0d4CE594B9F8E081cD4caB993be33D7ef35F30`
- Runtime bytecode hash: `0x28bb1dc472342c927d807fb08147fb6b1158a873f24d813b8440397d66f5b892`
- [Commit transaction](https://polygonscan.com/tx/0x4deed344b0e9a708ad53a84e9ce259c6f9b090e12de7c69930d72ad5a91b5a5f)
- [Reveal transaction](https://polygonscan.com/tx/0x1b6b9d1b5d0d2234e6a438816741aee1d09a1f86e132f9aefd6485e57253dfd8)
- [Resolve transaction](https://polygonscan.com/tx/0x4674f2a98151876f23baca2ae0f3aca19b3caa77bebce434945e78045d0b619c)

The proof-of-concept contract is ownerless and non-upgradeable, has no payable entry points or external calls, and stores no funds or trading permissions.

## Integration checklist

- Use Polygon chain ID 137 and verify the exact registry address and runtime bytecode.
- Fetch the current author nonce immediately before constructing the commitment.
- Use a fresh random salt for every commitment.
- Persist the full preimage privately and durably until reveal.
- Use canonical, versioned documents and unambiguous Polymarket identifiers.
- Wait for transaction confirmation before advancing lifecycle state.
- Reconcile claimed fills against confirmed trade/on-chain data before producing evidence.
- Keep official Builder attribution and independent provenance separate in UI and reporting.
- Make resolver identity, methodology, and withholding risk explicit.
- Provide an indexer or event scanner if users need searchable histories.
