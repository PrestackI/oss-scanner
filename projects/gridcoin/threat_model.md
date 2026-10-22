# Threat model: Gridcoin-Research


## What this project does and where untrusted input enters

Gridcoin-Research is the reference implementation of the Gridcoin proof-of-stake cryptocurrency (a Bitcoin and
Peercoin descendant, C++17, CMake). `gridcoinresearchd` is a full node and wallet; it also stakes blocks and rewards
BOINC volunteer computing through on-chain beacons and superblocks. Untrusted input enters at:

- **The P2P network (most important):** every message from a peer (`src/net.cpp`, `src/net_processing.cpp`,
  `src/main.cpp`): version handshake, inventory, blocks, transactions, headers and addresses. Peers are anonymous and
  adversarial.
- **Blocks and transactions under consensus rules:** validation of proof-of-stake blocks, coinstake and rewards,
  beacons, superblocks and their scraper convergence, polls and votes, and contract payloads (`src/validation*`,
  `src/gridcoin/`). Anything a remote peer can get past validation is consensus-critical.
- **The scraper** (`src/gridcoin/scraper/`): on nodes that run it, it downloads BOINC project statistics over HTTPS and
  parses them. The remote servers are third parties and their content is untrusted.
- **Files on disk:** `wallet.dat` (BerkeleyDB), the block files and LevelDB indexes, `peers.dat`, `banlist`, the
  config file. The local user is trusted, but a corrupted or truncated file must not lead to memory corruption.
- **JSON-RPC** (`src/rpc/`): authenticated with `-rpcuser`/`-rpcpassword`; by default it is bound to localhost.
  Post-authentication input is from a trusted operator, but parsing must still not corrupt memory.

## Components that matter most / least

- Most: P2P message handling, block and transaction validation, consensus code under `src/gridcoin/` (staking,
  rewards, beacons, superblocks, voting), the scraper's parsers, and wallet key handling and encryption.
- Less: RPC handlers (authenticated operator), the logging and archive code.
- Out of scope: the Qt GUI (`src/qt/`, not built in this image), `depends/` and bundled third-party code (`src/bdb53`,
  `src/leveldb`, `src/secp256k1`, `src/univalue`, `src/crc32c`) except where Gridcoin's own use of them is the bug.

## How to exercise it

- Unit tests: `/src/build/src/test/test_gridcoin` (Boost.Test; `--run_test=<suite>` to pick one).
- Functional tests: `python3 test/functional/test_runner.py --configfile=/src/build/test/config.ini` drives regtest
  nodes over RPC and P2P, local only; name one test (`... test_runner.py --configfile=... <name>.py`) to run it alone.
- Node: `/src/build/bin/gridcoinresearchd -regtest -datadir=<dir>`. Regtest needs no network and can mine
  blocks on demand (`generatetoaddress`).

## How you rate severity (proposal)

- **Critical:** a remote peer can cause a consensus split, accept an invalid block or reward, mint or steal coins, or
  achieve remote code execution; any memory corruption reachable from the P2P network or from a block.
- **High:** a remotely triggered crash or hang of a node (a single message or a block that crashes peers); wallet key
  or passphrase disclosure; scraper input that corrupts memory or the stats a superblock is built from.
- **Medium:** remote resource exhaustion that degrades but does not stop a node; memory-safety bugs reachable only
  from local files or authenticated RPC.
- **Low:** bugs that need a malicious local operator, or crashes on clean shutdown paths.

## Anything to leave alone

- Mainnet-only behaviour that cannot be reached on regtest or testnet should be reported with the reasoning, not
  probed against the live network.
- Do not report the use of `assert()` in itself; asserts are live in every build configuration (`-UNDEBUG`).
