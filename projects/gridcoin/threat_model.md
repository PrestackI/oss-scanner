# Threat model: Gridcoin-Research

## What this project does and where untrusted input enters

Gridcoin-Research is the reference implementation of the Gridcoin proof-of-stake cryptocurrency (a Bitcoin and
Peercoin descendant, C++17, CMake). `gridcoinresearchd` is a full node and wallet; it also stakes blocks and rewards
BOINC volunteer computing through on-chain beacons and superblocks. Untrusted input enters at:

- **The P2P network (most important):** every message from a peer (`src/net.cpp`, `src/net_processing.cpp`,
  `src/validation.cpp`): version handshake, inventory and getdata, blocks, transactions, getblocks/getheaders
  requests, addresses and pings; Gridcoin's own messages `scraperindex` and `part` (scraper manifests,
  `src/gridcoin/scraper/scraper_net.cpp`, received and parsed by every node, not only scrapers) and `psgt`
  (`src/psgt.cpp`); and the inherited signed `alert` message (`src/alert.cpp`). Gridcoin also accepts `aries`,
  `encrypt` and `gridaddr` as aliases for `version`, `block` and `addr`. Peers are anonymous and adversarial;
  any parsing that runs before a manifest's signature and authorization checks is reachable by any peer.
- **Blocks and transactions under consensus rules:** validation of proof-of-stake blocks, coinstake and rewards,
  beacons, superblocks and their scraper convergence, polls and votes, and contract payloads (`src/validation*`,
  `src/gridcoin/`). Anything a remote peer can get past validation is consensus-critical. Rules gated on V15 (the
  on-chain pool registry, the PSGT pool after decoding, the v15 manifest project cap) are merged but not yet scheduled
  on any network (`BlockV15Height` is unset in all three chain params). Exercise them on regtest with
  `-blockv15height=0`, as `p2p_psgt_relay.py` does, and say in the report that the code is not yet active.
- **The scraper's downloads** (`src/gridcoin/scraper/`): on nodes started with `-scraper` and authorized on chain,
  statistics files from the BOINC project servers on the on-chain whitelist. Their bytes are untrusted, but each
  server is authoritative for its own project's credit, so a server misreporting its own statistics is not a finding.
  Corrupting memory, crashing the scraper, or changing other projects' statistics or the convergence is. The scraper
  threads do not run under `-regtest`; reproduce through the unit tests (`src/test/gridcoin/scraper_tests.cpp`,
  `scraper_net_tests.cpp`).
- **The update check:** on mainnet, unless `-disableupdatecheck` is set, the node fetches GitHub's releases API
  (`src/gridcoin/scraper/http.cpp`) and parses the JSON reply (`src/gridcoin/upgrade.cpp`). It cannot run here;
  reproduce with a unit test.
- **Files on disk:** `wallet.dat` (BerkeleyDB), the block files and LevelDB indexes, `peers.dat`, `banlist.dat`, the
  config file. The local user is trusted, but a corrupted or truncated file must not lead to memory corruption.
- **JSON-RPC** (`src/rpc/`): authenticated with `-rpcuser`/`-rpcpassword`; it listens on loopback unless
  `-rpcbind` or `-rpcallowip` is set. Post-authentication input is from a trusted operator, but parsing must still
  not corrupt memory, and wallet RPCs must not lose the owner's funds.

## Components that matter most / least

- Most: P2P message handling, block and transaction validation, consensus code under `src/gridcoin/` (staking,
  rewards, beacons, superblocks, voting), the scraper's manifest handling and parsers, and wallet key handling and
  encryption.
- Less: RPC handlers (authenticated operator), debug-log rotation and archiving.
- Out of scope: the Qt GUI (`src/qt/`, not built in this image), `depends/` (not used by this build), and bundled
  third-party code (`src/bdb53`, `src/leveldb`, `src/secp256k1`, `src/univalue`, `src/crc32c`, `src/crypto/ctaes`,
  `src/ipc/libmultiprocess`, `src/test/xxd`, `src/gridcoin/md5.c`) except where Gridcoin's own use of it is the bug.

## How to exercise it

- Unit tests: `/src/build/src/test/test_gridcoin` (Boost.Test; `--run_test=<suite>` to pick one).
  `src/test/chain_setup.h` builds a regtest chain inside a unit test. A new test file must be listed in
  `src/test/CMakeLists.txt`; rebuild with `cmake --build /src/build --target test_gridcoin`.
- Functional tests: `python3 test/functional/test_runner.py --configfile=/src/build/test/config.ini` runs the suite
  (regtest, local only). To run one script alone, new or existing, call it directly:
  `python3 test/functional/<name>.py --configfile=/src/build/test/config.ini`. test_runner.py ignores any name not in
  its own script lists: it prints "No valid test scripts specified" and exits 0 without running anything.
  `test/functional/test_framework/p2p.py` sends crafted P2P messages; `p2p_psgt_relay.py` is an example.
- Node: `/src/build/bin/gridcoinresearchd -regtest -datadir=<dir>`. Regtest needs no network and can mine
  blocks on demand (`generatetoaddress`). There is no separate CLI: call RPC with the same binary, e.g.
  `gridcoinresearchd -regtest -datadir=<dir> getnewaddress`.
- Regtest keeps the wallet in memory and never reads `wallet.dat`; for a wallet-file reproducer use a unit test.

## How we rate severity

- **Critical:** an unintended consensus divergence between nodes enforcing the same rules, including across
  platforms or compilers, whether or not an attacker triggers it (a rule change at a scheduled activation height is
  intended, not a divergence); a remote peer can get an invalid block or reward accepted, mint or steal coins, or achieve
  remote code execution; a relayed block or transaction that crashes or stalls every node that processes it; any
  memory corruption reachable from the P2P network or from a block; wallet keys or passphrase disclosed to a remote
  party.
- **High:** a remotely triggered crash or hang of a single node; wallet keys or passphrase disclosed locally (logs,
  other local users); wallet logic that loses or misdirects the owner's funds in normal use, including through RPC;
  scraper input that corrupts memory or changes the statistics a superblock is built from.
- **Medium:** remote resource exhaustion that degrades but does not stop a node; memory-safety bugs reachable only
  from local files or authenticated RPC.
- **Low:** bugs that need a malicious local operator, or crashes on clean shutdown paths.

## What a report should show

- The untrusted input above that reaches the defect, and the path from it. If you could not complete the path, say
  where it stopped and mark the finding "path not demonstrated" rather than lowering its severity. Only a defect you
  have reason to believe no listed input reaches is at most Low.
- A reproducer that runs in this image: a unit test, a regtest functional test, or a crafted P2P message, block or
  file. A finding without one should say so, and why it could not be built.
- Trust assumptions, not vulnerabilities: administrative contracts signed with the network's master key (protocol
  entries, the project whitelist, scraper authorization) are trusted, and attacks that need a majority of stake
  weight are inherent to proof-of-stake. Report them only if a smaller actor can achieve the same effect.
- One report per root cause, listing every entry point it was reached from.

## Anything to leave alone

- Mainnet-only paths cannot be reached here; reason about them and say so rather than guessing at live behaviour.
- Regtest is test scaffolding, not a weaker mainnet. Under `-regtest` (`Params().IsMockableChain()`) the node skips
  coinbase and coinstake maturity, honours the consensus overrides in `CONSENSUS_OVERRIDE_ARGS` (`src/chainparams.h`,
  gated by `ConsensusOverridesAllowed()`), and offers regtest-only RPCs (`generate`, `generatetoaddress`,
  `generatesuperblock`, ...); `generatesuperblock` builds a superblock, and can activate beacons, with no scraper
  convergence. A behaviour that exists only because of one of these is not a finding: confirm that the mainnet branch
  of the same code shows it too. A relaxation that can be reached off regtest (for example, a gate that checks the
  wrong condition) is a finding.
- `assert()`: asserts are compiled into every build of this tree, so a peer-reachable assert failure is a remote crash
  and is rated as one. An assert whose condition no listed input can falsify is not a finding. An assert whose
  argument has side effects the program relies on may be reported as Low.
- A UBSan unsigned-integer-overflow or implicit-conversion report is not a bug by itself (CI builds with
  `-fsanitize=integer`); report it only with the wrong behaviour it causes.
