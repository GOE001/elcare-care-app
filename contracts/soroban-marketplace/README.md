<div align="center">

# soroban-marketplace

**Core marketplace smart contract for ElcareHub — built with Rust and the Soroban SDK on Stellar.**

</div>

---

## Table of Contents

- [Overview](#overview)
- [Contract Functions](#contract-functions)
- [Contract Versioning & Migration](#contract-versioning--migration)
- [Price Bounds](#price-bounds)
- [Data Types](#data-types)
- [Storage Layout](#storage-layout)
- [Error Codes](#error-codes)
- [Prerequisites](#prerequisites)
- [Build](#build)
- [Test](#test)
- [Deploy](#deploy)
- [Manual Invocation](#manual-invocation)

---

## Overview

This contract manages the complete lifecycle of on-chain marketplace listings, auctions, and offers. All state lives in **Soroban persistent storage** — no database is needed for the contract itself. The off-chain indexer reads emitted events to reconstruct a queryable view.

**What the contract handles:**
- NFT listings with multi-recipient royalty splits
- Fixed-price sales with whitelisted token support
- Auctions with reserve prices, bidding, and finalization
- Offer system — make, accept, reject, withdraw
- Protocol fee collection to a configurable treasury
- Admin controls — pause/unpause, token whitelist, artist revocation
- Upgradable-contract version discovery via `version()`
- Admin-guarded, idempotent storage migration via `migrate()`
- Global price bounds to prevent dust listings and overflow-risk prices

---

## Contract Functions

### Listings

| Function | Auth | Description |
|----------|------|-------------|
| `create_listing(artist, metadata_cid, collection, token_id, price, currency, token, recipients)` | artist | Creates a listing, returns `listing_id` |
| `update_listing(artist, listing_id, new_price)` | artist | Updates price of an active listing |
| `cancel_listing(artist, listing_id)` | artist | Cancels an active listing |
| `buy_artwork(buyer, listing_id)` | buyer | Purchases listing, distributes payment + royalties |
| `get_listing(listing_id)` | — | Returns full `Listing` struct |
| `get_total_listings()` | — | Total listing count |
| `get_artist_listings(artist)` | — | `Vec<u64>` of artist's listing IDs |

### Auctions

| Function | Auth | Description |
|----------|------|-------------|
| `create_auction(creator, token, collection, token_id, reserve_price, duration, recipients)` | creator | Creates an auction. `duration` must be ≥ `MIN_AUCTION_DURATION` (3 600 s / 1 hour); rejects with `InvalidAuctionDuration` otherwise. |
| `place_bid(bidder, auction_id, bid_amount)` | bidder | Places a bid above the current highest. Appends a `BidRecord` to the bounded on-chain history (capped to `BID_HISTORY_CAP = 20`). |
| `finalize_auction(caller, auction_id)` | anyone | Finalizes after `end_time` — transfers NFT to winner and distributes funds. |
| `cancel_auction(creator, auction_id)` | creator | Cancels a no-bid auction before it ends. |
| `get_auction(auction_id)` | — | Returns full `Auction` struct. |
| `get_auction_bids(auction_id)` | — | Returns the bounded bid history (`Vec<BidRecord>`) in chronological order (oldest → newest). At most `BID_HISTORY_CAP` (20) entries are retained; older bids are evicted. Returns an empty vec when no bids have been placed. |

### Offers

| Function | Auth | Description |
|----------|------|-------------|
| `make_offer(offerer, listing_id, amount, token)` | offerer | Makes an offer on a listing |
| `accept_offer(artist, listing_id, offer_id)` | artist | Accepts an offer, marks listing Sold |
| `reject_offer(artist, listing_id, offer_id)` | artist | Rejects an offer |
| `withdraw_offer(offerer, offer_id)` | offerer | Withdraws a pending offer |

### Admin

| Function | Auth | Description |
|----------|------|-------------|
| `initialize(admin, treasury, fee_bps)` | — | One-time setup |
| `set_admin(new_admin)` | admin | Immediate admin transfer |
| `propose_admin(proposed)` | admin | Step 1 of 2-step transfer |
| `accept_admin()` | proposed | Step 2 of 2-step transfer |
| `pause()` / `unpause()` | admin | Circuit breaker — blocks all state changes |
| `add_token(token)` / `remove_token(token)` | admin | Manage payment token whitelist |
| `revoke_artist(artist)` / `reinstate_artist(artist)` | admin | Artist access control |
| `set_treasury(address)` / `set_fee_bps(bps)` | admin | Update protocol fee config |
| `version()` | — | Returns the current contract semantic version |
| `migrate(admin)` | admin | Idempotent storage migration for upgrades |
| `set_price_bounds(admin, min, max)` | admin | Set global min/max listing price |
| `get_price_bounds()` | — | Returns `(Option<i128>, Option<i128>)` |

---

## Contract Versioning & Migration

### Overview

The contract implements a discoverable version and a guarded migration path to support forward-compatible upgrades when storage shape changes are introduced.

### `version() → String`

Returns the current **semantic version** of the deployed contract binary (e.g. `"1.0.0"`). This is a constant baked into the WASM — it never changes after deployment. Use it in upgrade scripts to verify you are interacting with the expected binary version before calling `migrate`.

```bash
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $STELLAR_SECRET \
  --network testnet \
  -- version
# → "1.0.0"
```

### `migrate(admin) → ()`

An **admin-guarded, idempotent** entry point reserved for storage migrations.

**How it works:**
1. Requires `admin` authorization and verifies the caller is the stored admin.
2. Looks up a per-version migration marker in persistent storage (`DataKey::MigrationDone("1.0.0")`).
3. If the marker already exists, **reverts with `AlreadyMigrated` (#29)** — this prevents double-application of the same migration.
4. Executes any version-specific storage back-fill logic (see the `contract.rs` source for per-version blocks).
5. Writes the marker to make the migration permanent.

**Idempotency guarantee:** Calling `migrate` twice for the same version always reverts on the second call, regardless of admin changes between calls. The marker is keyed by the version string, not by the admin address.

### Migration policy

| Contract version | Storage changes | Migration notes |
|-----------------|-----------------|-----------------|
| `1.0.0` | None (baseline) | Marker written; future versions back-fill from here |

**Adding a migration for a future version:**
1. Bump `CONTRACT_VERSION` in `contract.rs` to the new semver string.
2. Add a per-version block inside `migrate()`:
   ```rust
   // Inside migrate():
   // if CONTRACT_VERSION == "1.1.0" {
   //     // e.g. back-fill new `min_price` field on existing listings
   // }
   ```
3. Deploy the new WASM and invoke `migrate(admin)` once.

**Upgrade script example:**
```bash
# 1. Deploy new WASM
stellar contract install --wasm target/wasm32v1-none/release/soroban_marketplace.wasm
stellar contract upgrade --id $CONTRACT_ID --wasm-hash $WASM_HASH

# 2. Confirm version
stellar contract invoke --id $CONTRACT_ID -- version
# → "1.1.0"

# 3. Run migration (admin-only, idempotent)
stellar contract invoke --id $CONTRACT_ID --source $ADMIN_SECRET -- migrate --admin $ADMIN_PUBLIC

# 4. Verify migration is marked (second call should fail with AlreadyMigrated)
stellar contract invoke --id $CONTRACT_ID --source $ADMIN_SECRET -- migrate --admin $ADMIN_PUBLIC
# → Error(Contract, #29) AlreadyMigrated — expected
```

---

## Price Bounds

### Overview

Admins can set a global `[min_price, max_price]` range that all new listings and auction reserve prices must fall within. This prevents:
- **Dust listings** — spam listings at price `1` that clog the active index.
- **Overflow-risk prices** — absurdly large prices that could cause integer overflow in payout math.

**Backward compatibility:** Price bounds are enforced only on *new* items created after the bounds are set. Existing listings and auctions are not retroactively affected.

**Permissive defaults:** When no bounds are set (fresh deploy or bounds never configured), all positive prices are accepted — identical to the pre-bounds behavior.

### `set_price_bounds(admin, min, max)`

Sets both the global minimum and maximum price in a single atomic call.

**Validation:**
- `min` and `max` must both be ≥ 0.
- `min` must be ≤ `max`. Violating this reverts with `InvalidPrice` (#2).
- Only the stored admin may call this. Non-admin callers revert with `Unauthorized` (#5).

```bash
# Set min = 10_000 stroops, max = 1_000_000_000_000 stroops
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $ADMIN_SECRET \
  --network testnet \
  -- set_price_bounds \
  --admin $ADMIN_PUBLIC \
  --min 10000 \
  --max 1000000000000
```

### `get_price_bounds() → (Option<i128>, Option<i128>)`

Returns the current `(min_price, max_price)` tuple. `None` means that bound is not configured (no limit in that direction).

```bash
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $STELLAR_SECRET \
  --network testnet \
  -- get_price_bounds
# → (Some(10000), Some(1000000000000))
```

### Enforcement

| Entry point | Check |
|-------------|-------|
| `create_listing` | `price` must be within `[min_price, max_price]` |
| `create_auction` | `reserve_price` must be within `[min_price, max_price]` |

Violations revert with `PriceOutOfBounds` (#30).

---

## Data Types

```rust
pub struct Listing {
    pub listing_id:   u64,
    pub artist:       Address,
    pub metadata_cid: String,        // IPFS CID of artwork metadata JSON
    pub collection:   Address,       // NFT collection contract
    pub token_id:     u64,           // Token ID within the collection
    pub price:        i128,          // in stroops (1 XLM = 10_000_000)
    pub currency:     String,        // "XLM" or token symbol
    pub token:        Address,       // Payment token contract address
    pub recipients:   Vec<Recipient>, // Royalty split
    pub status:       ListingStatus, // Active | Sold | Cancelled
    pub owner:        Option<Address>,
    pub created_at:   u32,           // Ledger sequence number
}

pub struct Recipient {
    pub address:    Address,
    pub percentage: u32,             // Basis points (10000 = 100%)
}

pub enum ListingStatus { Active, Sold, Cancelled }

pub struct Auction {
    pub auction_id:     u64,
    pub creator:        Address,
    pub collection:     Address,
    pub token_id:       u64,
    pub token:          Address,
    pub reserve_price:  i128,
    pub highest_bid:    i128,
    pub highest_bidder: Option<Address>,
    pub end_time:       u64,         // Ledger sequence
    pub status:         AuctionStatus,
    pub recipients:     Vec<Recipient>,
    pub created_at:     u32,
}

pub enum AuctionStatus { Active, Finalized, Cancelled }

/// A single entry in the per-auction bounded bid history.
pub struct BidRecord {
    pub bidder: Address,   // Account that placed this bid
    pub amount: i128,      // Bid amount in payment-token stroops
    pub ledger: u32,       // Ledger sequence number when the bid was recorded
}
```

### Constants

| Constant | Value | Description |
|----------|-------|-------------|
| `MIN_AUCTION_DURATION` | `3_600` seconds (1 hour) | Minimum `duration` accepted by `create_auction`. Auctions shorter than this are rejected with `InvalidAuctionDuration (#31)` to prevent meaningless or front-runnable auctions. |
| `BID_HISTORY_CAP` | `20` entries | Maximum number of `BidRecord` entries retained per auction on-chain. When a new bid is placed and the history is already full, the oldest entry is evicted. Use the off-chain indexer for full bid history beyond this cap. |

---

## Storage Layout

```
Persistent key                           Value
──────────────────────────────────────────────────────────────
DataKey::ListingCount                    u64
DataKey::Listing(listing_id: u64)        Listing
DataKey::ArtistListings(Address)         Vec<u64>
DataKey::AuctionCount                    u64
DataKey::Auction(auction_id: u64)        Auction
DataKey::AuctionBids(auction_id: u64)    Vec<BidRecord>  (capped to BID_HISTORY_CAP=20)
DataKey::OfferCount                      u64
DataKey::Offer(offer_id: u64)            Offer
DataKey::Admin                           Address
DataKey::PendingAdmin                    Address
DataKey::Treasury                        Address
DataKey::ProtocolFeeBps                  u32
DataKey::MinBidIncrement                 i128
DataKey::AuctionExtensionWindow          u64
DataKey::AuctionExtensionTrigger         u64
DataKey::IsPaused                        bool
DataKey::TokenWhitelist                  Vec<Address>
DataKey::RevokedArtist(Address)          bool
DataKey::ActiveListings                  Vec<u64>
```

All persistent entries use `extend_ttl` on every read/write (~30-day TTL via `LEDGER_TTL_THRESHOLD = 144_000`, `LEDGER_TTL_BUMP = 432_000`).

---

## Error Codes

| Code | Value | Meaning |
|------|-------|---------|
| `InvalidCid` | 1 | Listing CID validation failure (legacy) |
| `InvalidPrice` | 2 | Price / fee amount is invalid (≤ 0 or > limit) |
| `ListingNotFound` | 3 | Listing ID does not exist |
| `ListingNotActive` | 4 | Listing is Sold or Cancelled |
| `Unauthorized` | 5 | Caller does not have required auth |
| `CannotBuyOwnListing` | 6 | Artist cannot purchase their own listing |
| `InvalidSplit` | 7 | Recipient array is empty |
| `TooManyRecipients` | 8 | More than 4 recipients provided |
| `AuctionNotFound` | 9 | Auction ID does not exist |
| `AuctionNotActive` | 10 | Auction is already finalized or cancelled |
| `BidTooLow` | 11 | Bid below reserve price or min-increment threshold |
| `AuctionExpired` | 12 | Bid placed after `end_time` |
| `AuctionNotExpired` | 13 | (reserved) |
| `AuctionAlreadyFinalized` | 14 | Auction already settled; cannot finalize or cancel again |
| `ArtistRevoked` | 15 | Revoked artist attempted a creation action |
| `OfferNotFound` | 16 | Offer ID does not exist |
| `CannotOfferOwnListing` | 17 | Artist cannot make an offer on their own listing |
| `OfferNotPending` | 18 | Offer is not in Pending state |
| `InsufficientOfferAmount` | 19 | Offer amount ≤ 0 |
| `ListingSold` | 20 | Listing is already sold |
| `ListingCancelled` | 21 | Listing is already cancelled |
| `ReentrancyGuard` | 22 | Re-entrant call detected on the same listing/auction |
| `ContractPaused` | 23 | Contract is paused by admin |
| `InvalidRoyalty` | 24 | Royalty bps > 10 000 |
| `TokenNotWhitelisted` | 25 | Payment token removed from whitelist since listing creation |
| `RoyaltyExceedsLimit` | 26 | Sum of recipient bps + protocol fee > 10 000 |
| `ListingExpired` | 27 | Listing `expires_at` has passed |
| `ListingNotExpired` | 28 | `expire_listing` called before the expiry timestamp |
| `AuctionNotEnded` | 28 | `finalize_auction` called before `end_time` |
| `AuctionHasBids` | 30 | `cancel_auction` called on an auction with an active highest bidder |
| `InvalidAuctionDuration` | 31 | `create_auction` `duration` < `MIN_AUCTION_DURATION` (3 600 s) |
| `SelfBidNotAllowed` | 32 | `place_bid` called by the auction creator (shill-bid prevention) |

---

## Prerequisites

```bash
# 1. Rust toolchain
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup target add wasm32v1-none

# 2. Stellar CLI
cargo install --locked stellar-cli --features opt
```

---

## Build

```bash
make build
# or directly:
cargo build --target wasm32v1-none --release
```

Output: `target/wasm32v1-none/release/soroban_marketplace.wasm`

Optimise WASM size (strips dead code):

```bash
make optimize
# or:
stellar contract optimize --wasm target/wasm32v1-none/release/soroban_marketplace.wasm
```

---

## Test

```bash
make test
# with output:
make test-verbose
# or directly:
cargo test
```

All tests use `Env::default()` with `mock_all_auths()` — no live network or wallet needed.

---

## Deploy

```bash
cd ../../scripts/deploy
./fund_account.sh        # fund test keypair
./deploy_contract.sh     # build + deploy + print CONTRACT_ID
```

---

## Manual Invocation

```bash
# Source deployment env vars
source ../../scripts/deploy/.env.deploy

# Create a listing
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $STELLAR_SECRET \
  --network testnet \
  -- create_listing \
  --artist $STELLAR_PUBLIC \
  --metadata_cid "QmYourIPFSCIDHere" \
  --price 10000000 \
  --currency XLM

# Query total listings
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $STELLAR_SECRET \
  --network testnet \
  -- get_total_listings

# Query contract version
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $STELLAR_SECRET \
  --network testnet \
  -- version

# Run storage migration (admin only, idempotent)
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $ADMIN_SECRET \
  --network testnet \
  -- migrate \
  --admin $ADMIN_PUBLIC

# Set global price bounds
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $ADMIN_SECRET \
  --network testnet \
  -- set_price_bounds \
  --admin $ADMIN_PUBLIC \
  --min 10000 \
  --max 1000000000000

# Pause the contract (admin only)
stellar contract invoke \
  --id $CONTRACT_ID \
  --source $ADMIN_SECRET \
  --network testnet \
  -- pause
```