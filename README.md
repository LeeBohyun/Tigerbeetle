# How to Run TigerBeetle

TigerBeetle is a distributed financial accounting database designed for mission-critical workloads.  
This guide shows how to build and run benchmarks locally.  

Repository: [tigerbeetle/tigerbeetle](https://github.com/tigerbeetle/tigerbeetle)

---

## Prerequisites
- Linux (x86_64)
- [Zig compiler](https://ziglang.org/download/) (v0.13.0 or newer)
- Root access (for benchmarking with raw devices)

---

## 1. Install Zig

Check the latest release at [ziglang.org/download](https://ziglang.org/download/).  
Example for v0.13.0:

```bash
cd ~
wget https://ziglang.org/download/0.13.0/zig-linux-x86_64-0.13.0.tar.xz
tar xf zig-linux-x86_64-0.13.0.tar.xz
echo 'export PATH="$HOME/zig-linux-x86_64-0.13.0:$PATH"' >> ~/.bashrc
source ~/.bashrc
zig version
```

## 2. Clone and Build TigerBeetle
```bash
git clone https://github.com/tigerbeetle/tigerbeetle.git
cd tigerbeetle

# Build with system Zig
zig build -Drelease=true
```

or use a bundled Zig version:
```bash
# Download the compiler
bash zig/download.sh

# Build with verification enabled
./zig/zig build --release -Dconfig_verify=true
```
## 3. Run Benchmark
```bash
# Wipe the target device
sudo blkdiscard /dev/nvme1n1

# Run benchmark
./tigerbeetle benchmark \
  --cache-grid=32GiB \
  --transfer-count=400000  --account-count=100000\
  --file=/dev/nvme1n1
```

## Benchmark workload analysis
tigerbeetle provides a synthetic benchmark as shown above.
It runs a scripted workload against a TigerBeetle cluster using N client instances. 
The workload is intentionally valid-only (no failed ops): accounts and transfers are constructed so the state machine accepts them. 
The benchmark measures per-request latency and throughput, prints percentile latencies, and (optionally) verifies that reads match what was written.

- There are two regions: ``ccount_count_hot`` are hot, and the rest are cold.
- whenever the benchmark needs an account index, it uses the chosen distribution (``--account-distribution``, either ``zipfian``/``latest``/``uniform``) within the selected region
- For transfers: with probability ``transfer_hot_percent%``, the debit account is chosen from hot; otherwise (the other ~%), both debit and credit are chosen from cold.
- The credit side is always chosen from the cold set, and it’s forced to be different from the debit.
- For queries (``get_account_transfers``), it always queries a hot account, so result sizes stay comparable run-to-run.
- create_transfers: write/update (creates transfer records; this mutates ledger state in the TB state machine—balances, pending timeouts, etc.—so effectively an update to system state).
- get_account_transfers: read/lookup (fetches transfers for an account).
- validate_accounts / validate_transfers (optional): read/lookup (does lookup_accounts and lookup_transfers to verify fields match what was written).
- There are no deletes and no failing ops; all data is crafted to be valid.

1. Account ID selection (sequential/random/reversed) -> and send create_accounts request
2. create_transfers stage: Which existing account will be the debit/credit side of this transfer?
3. The get_account_transfers stage selects a hot account ID to read its transfers.
4. The validate_accounts stage regenerates the same IDs and checks that they match the ones stored.
 

### Step 1. create_accounts (isnerts)
- ``insert_account*(tb.Account)``
```bash
for i in 1..account_count:
  id = permute_id(i)                  // sequential/random/reversed, per CLI
  acct = {
    id,
    user_data_128 = rand128(),
    user_data_64  = rand64(),
    user_data_32  = rand32(),
    ledger = 2, code = 1,
    flags = { history = flag_history, imported = flag_imported },
    timestamp = flag_imported ? i : 0,
    debits_pending = debits_posted = credits_pending = credits_posted = 0
  }
  insert_account(acct)                // ← create_accounts write

```

### Step 2. create_transfers (update)
- Select account index based on hot/cold selection logic:
```bash
function choose_account_index(region):
  if region == HOT:
    // sample within [1..account_count_hot] using the chosen distribution
    return sample_hot_index()                         // e.g., 17
  else:
    // sample within [account_count_hot+1..account_count] using the distribution
    return sample_cold_index()                        // e.g., 534
```

- And create one transfer
```bash
// 70%: debit from hot, credit from cold; else both from cold
if rand_percent() <= transfer_hot_percent:
  debit_idx  = choose_account_index(HOT)              // e.g., 17
  credit_idx = choose_account_index(COLD)             // e.g., 534
else:
  debit_idx  = choose_account_index(COLD)             // e.g., 921
  credit_idx = choose_account_index(COLD)             // e.g., 534

if credit_idx == debit_idx:
  credit_idx = (credit_idx % account_count) + 1       // ensure different

debit_id  = permute_id(debit_idx)
credit_id = permute_id(credit_idx)

pending   = transfer_pending && chance(3/10)          // ~30% pending
timeout   = pending ? rand_int_inclusive(1, 5) : 0

transfer = {
  id = permute_id(next_transfer_seq()),
  debit_account_id  = debit_id,
  credit_account_id = credit_id,
  amount  = random_exponential_u64(max=10_000) + 1,
  code    = rand_u16_nonzero(),
  flags   = { pending, imported = flag_imported },
  timeout,
  user_data_128 = rand128(), user_data_64 = rand64(), user_data_32 = rand32(),
  ledger = 2,
  timestamp = flag_imported ? (account_index + transfer_index + 1) : 0
}

insert_transfer(transfer)                              // ← create_transfers write
// (This “write” mutates the ledger state: balances, pending queues, etc.)
```

### Step 3. get_account_transfers (reads/lookups)
- always query hot to stabilize result sizes:
- ``results = query_account_transfers(account_id, filter)``
  
```bash
for q in 1..query_count:
  account_idx = choose_account_index(HOT)             // e.g., 23
  account_id  = permute_id(account_idx)
  results = query_account_transfers(
              account_id,
              flags = { credits = true, debits = true, reversed = false },
              limit  = max_that_fits_in_message)
  // sanity: for each t in results, exactly one side matches account_id
  assert( (t.debit_account_id == account_id) XOR (t.credit_account_id == account_id) )
```

### Step 4. validate_accounts (lookups, optional (?))
```bash
for each batch of expected_ids:
  ids = [ permute_id(i) for i in batch_indices ]
  db_accounts = lookup_accounts(ids)                  // ← read
  assert_fields_equal(expected_accounts(batch), db_accounts)
```

### Step 5. validate_transfers (lookups, optional)
```bash
for each batch of expected_transfer_ids:
  ids = [ permute_id(transfer_seq) for seq in batch ]
  db_transfers = lookup_transfers(ids)                // ← read
  assert_fields_equal(expected_transfers(batch), db_transfers)
```
