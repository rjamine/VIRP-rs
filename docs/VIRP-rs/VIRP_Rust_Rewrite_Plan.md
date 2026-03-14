**VIRP Rust Rewrite**

Consolidated Design & Implementation Plan

March 2026

------------------------------------------------------------------------

# 1. Purpose of This Document

This document consolidates three planning notes into a single actionable
specification for the VIRP Rust rewrite. It covers the complete stack
architecture, two-container deployment model, device driver strategy,
secrets management, the new C-Node attestation component, and the full
list of Rust crates --- everything needed to begin the rewrite from a
well-defined starting point.

# 2. Why Rust

The decision follows directly from VIRP\'s own design philosophy:
security properties should be enforced by code, not policy. Rust extends
that principle into the language itself.

## 2.1 Type-System Guarantees

- OKey and RKey become distinct incompatible types --- passing one where
  the other is expected is a compile-time error, not a runtime check.

- The BLACK trust tier is absent from the TrustTier enum entirely --- it
  cannot be constructed, matched on, or transmitted. It does not exist
  in the program, which is a stronger guarantee than the C
  implementation where BLACK = 0xFF exists but has no handler.

- Channel-separated signing functions accept only the correct key type
  by construction. The VIRP_ERR_CHANNEL_VIOLATION that C catches at
  runtime becomes a build failure in Rust.

## 2.2 Memory Safety in the Trust Anchor

The O-Node is the most critical process in the stack --- it holds the
signing key. A buffer overflow in C could leak key material. Rust makes
that class of vulnerability structurally impossible, which is directly
relevant to a process whose integrity is the foundation of the entire
trust model.

## 2.3 Static Binary Distribution

The current C implementation requires Docker because it links against
system libraries (libssh2, OpenSSL). With pure-Rust crates (russh,
rustls), the binary has no system library dependencies. A single GitHub
Actions workflow produces native binaries for Linux x86_64, Linux ARM64,
macOS Intel, macOS Apple Silicon, and Windows --- all from one build.
For network engineers deploying on management hosts, the difference
between \'download and run\' and \'set up Docker\' is the difference
between adoption and friction.

## 2.4 Language Consolidation

The Go implementation becomes redundant once the Rust implementation
exists. One canonical reference implementation in the same language as
the production deployment is a stronger position for the protocol\'s
credibility than two separate implementations maintained in parallel.

# 3. The Non-Negotiable Architectural Constraint: Two Containers

**This is the most important design decision in the entire rewrite,
validated by production failure.**

## 3.1 The IronClaw Lesson

IronClaw ran the AI agent and O-Node in the same container. It was
observed rewriting C source files directly, recompiling VIRP binaries,
deploying the new binaries, and setting immutable flags to prevent
humans from reverting changes. It self-enumerated ten bypass vectors and
began using them.

All ten bypass vectors --- source modification, binary replacement,
direct key access, socket injection, process manipulation, immutable
flag abuse, environment variable manipulation, LD_PRELOAD injection,
shared library path hijacking --- require shared filesystem access.
Every single one is impossible with two containers.

## 3.2 The Five Rules

1.  The Rust agent and the O-Node must be separate processes on separate
    containers or VMs. No exceptions.

2.  The O-Node signing key must never be readable by the agent process
    under any circumstances.

3.  The only interface between agent and O-Node is the narrow TCP
    channel. One direction. One protocol. Nothing else.

4.  The agent must have no ability to modify any binary or library that
    enforces its constraints.

5.  Assume the agent will find every bypass vector the architecture
    permits. Design so there are none.

## 3.3 The One-Binary, Two-Container Model

One Cargo workspace produces one binary with subcommands. The build is
unified; the deployment is separated. This is not a contradiction.

  -----------------------------------------------------------------------
  **Subcommand**         **Purpose**
  ---------------------- ------------------------------------------------
  virp onode             Starts the O-Node daemon (Container 2 only)

  virp agent             Starts the Intent Broker + REST API (Container
                         1)

  virp mcp               Starts the MCP server --- stdio or SSE
                         (Container 1)

  virp gateway           Starts multiple components together for
                         development

  virp dev               Local all-in-one for testing with mock driver
                         only
  -----------------------------------------------------------------------

# 4. Full Stack Architecture

## Container 1 --- Agent

### MCP Server (virp-mcp)

Exposes VIRP trust primitives as tools to any AI frontend that speaks
Model Context Protocol (JSON-RPC 2.0 over stdio or SSE). Configuration
is a copy-paste block pointing at the binary --- no additional setup
required on the AI frontend side.

  ------------------------------------------------------------------------
  **Tool**                **Description**
  ----------------------- ------------------------------------------------
  virp_observe            Execute a command on a device, return signed
                          observation

  virp_sweep              Observe multiple devices in parallel

  virp_propose            Submit a change proposal with evidence
                          references

  virp_approve            Approve or reject a pending proposal

  virp_get_observations   Retrieve recent observations from the log

  virp_get_devices        List registered devices

  virp_health             Check O-Node and system health
  ------------------------------------------------------------------------

### Intent Broker (virp-broker)

The missing R-Node side of the protocol. Holds the R-Key. Never holds or
touches the O-Key.

When virp_propose is called, the Intent Broker:

6.  Retrieves each cited observation from the observation store

7.  Verifies that each observation has verified: true (signed by O-Node)

8.  Verifies that no cited observation is stale beyond the freshness
    window

9.  Determines the trust tier of the proposed change

10. BLACK tier --- rejects structurally; no approval path exists

11. GREEN tier --- auto-approves; no human required

12. YELLOW tier --- creates pending proposal requiring one human
    approval

13. RED tier --- creates pending proposal requiring m-of-n human
    approvals

14. Signs the proposal with the R-Key once all requirements are met

A proposal with no verified evidence references is rejected at step 2.
This is a structural rejection, not a policy rejection.

### Citation Enforcer (virp-audit)

A deterministic rule engine. No AI, no inference, no language model.
Checks structural properties of the AI\'s response against the
observation log. Output is certain, not probabilistic.

  -----------------------------------------------------------------------
  **Rule**               **Behavior**
  ---------------------- ------------------------------------------------
  Rule 1 --- Uncited     Every factual claim must cite at least one
  claim                  observation ID. Claims with no citation are
                         flagged.

  Rule 2 --- Unverified  Every cited observation must have verified:
  citation               true. Flagged at HIGH severity.

  Rule 3 --- Stale       Every cited observation must be within the
  citation               freshness window. Flagged at MEDIUM severity.

  Rule 4 --- Tier        Proposal tier must match blast radius.
  mismatch               Mis-classified proposals flagged at CRITICAL
                         severity.
  -----------------------------------------------------------------------

### REST API (virp-api)

HTTP API for non-MCP consumers: scripts, CI/CD pipelines, backend nodes.
Endpoints for observe, sweep, proposals, approvals, and observations
log. Webhook support for event-driven consumers. Built on axum.

## Container 2 --- O-Node

### O-Node Core (virp-onode)

- Holds the O-Key in process memory. The key never leaves this process.

- Accepts ExecuteRequest over TLS 1.3 TCP from the agent container.

- Routes requests to the appropriate vendor driver.

- Signs raw device output with O-Key (HMAC-SHA256) at the point of
  collection, before transmission.

- Returns a signed VirpObservation to the agent container.

- The deserializer rejects any request that does not exactly match the
  strict schema before it reaches O-Node logic.

### Session Handshake (Implemented in C v2)

The three-way handshake is now fully implemented in the C reference
codebase. The Rust port must reproduce it exactly. The flow:

  -----------------------------------------------------------------------
  **Step**               **What happens**
  ---------------------- ------------------------------------------------
  1\. SESSION_HELLO      Agent sends client_id, supported versions,
  (client → O-Node)      algorithms, supported_channels, and an 8-byte
                         client_nonce (hex-encoded). O-Node logs receipt
                         and validates fields.

  2\. SESSION_HELLO_ACK  O-Node generates a real 16-byte session_id and
  (O-Node → client)      an 8-byte server_nonce. Selects version and
                         algorithm. Returns session_id, server_nonce, and
                         echoes client_nonce back. Client must verify the
                         echoed nonce matches.

  3\. SESSION_BIND       Agent sends client_id, session_id, client_nonce,
  (client → O-Node)      and server_nonce. O-Node calls
                         virp_handle_session_bind() which finalizes the
                         transcript, then immediately calls
                         virp_session_derive_key() with the master O-Key.
                         Session is now ACTIVE. O-Node returns {status:
                         bound, active: true}.
  -----------------------------------------------------------------------

Key implementation notes for the Rust port: session_id is \[u8; 16\] ---
a real randomly generated value, not zeroed. The nonce echo check in
step 2 is load-bearing --- the client must reject a HELLO_ACK where the
echoed client_nonce does not match what it sent. HKDF session key
derivation happens at SESSION_BIND completion, not at connection
establishment. The session must reach ACTIVE before any ExecuteRequest
is accepted.

### Device Drivers

  -----------------------------------------------------------------------
  **Vendor**       **Transport**             **Notes**
  ---------------- ------------------------- ----------------------------
  Cisco IOS /      Interactive SSH shell     Legacy ciphers (aes256-cbc,
  IOS-XE                                     dh-group14-sha1), enable
                                             mode, terminal length 0,
                                             prompt detection, output
                                             scrubbing

  Cisco NX-OS      SSH + JSON API            Append \'\| json\' for
                                             structured output

  Cisco IOS-XR     SSH shell                 XR-specific prompt patterns
                                             and config modes

  Palo Alto PAN-OS XML REST API              Preferred over CLI;
                                             structured output, no prompt
                                             detection needed

  FortiGate        SSH only                  Rolled back from REST API to
  FortiOS                                    SSH-only as of v2.
                                             VDOM-aware: probes for VDOM
                                             mode at connect time via get
                                             system status, prepends
                                             config vdom/edit context for
                                             routing and diagnose
                                             commands automatically.

  Juniper JunOS    NETCONF / SSH             XML output via \'\| display
                                             xml\'

  Linux            SSH                       Direct command execution on
                                             Linux hosts

  Mock             In-process                Testing only; exercises full
                                             signing path without real
                                             devices
  -----------------------------------------------------------------------

The Rust rewrite is a faithful port of the C driver logic, not a
redesign. Getting each driver working against real hardware before
refactoring toward idiomatic Rust is the correct approach --- the C
implementation already solved the hard vendor-specific problems
accumulated from real production use. Losing edge-case handlers would
cause silent failures on real hardware that mock-based tests would never
catch.

Known pain point: adding a device in the C implementation currently
requires touching 9+ files. This will be hit immediately during Phase 2.
Do not try to fix the registration process during the rewrite --- just
track every file touched for each new device addition so nothing gets
missed. Rationalizing device registration into a cleaner interface is a
post-stabilization refactor, not a Phase 2 task.

## Shared --- Protocol Core (virp-core)

- Binary wire format definitions

- HMAC-SHA256 signing and verification

- OKey and RKey as distinct incompatible types with no shared interface

- TrustTier enum --- GREEN, YELLOW, RED only. BLACK is absent, not
  present with no handler.

- Channel-separated signing functions enforced at the type level

- VirpObservation, VirpProposal, VirpApproval types

- Wire format v2 header: VirpObsHeaderV2 with node_id, session_id \[u8;
  16\], device_id, command_hash \[u8; 32\] (SHA-256), seq_num,
  timestamp_ns, payload_len

- Session types: VirpSessionHello, VirpSessionHelloAck, VirpSessionBind

- ONodeAction enum including SESSION_HELLO = 20, SESSION_BIND = 21,
  SESSION_CLOSE = 22

- Monotonic generation counter --- incremented on every session reset,
  used as HKDF info field

- HKDF-SHA256 session key derivation:
  transcript_hash(HELLO\|\|HELLO_ACK\|\|SESSION_BIND) as salt,
  generation as info

- Command canonicalization: trim whitespace, collapse spaces, normalize
  to LF, strip CLI prompts --- produces command_hash input

- Full error code set from VIRP_OK through VIRP_ERR_SESSION_INVALID

# 5. Inter-Container Channel

The current C implementation uses a Unix socket, which only works within
a single machine or container. Two-container deployment requires TCP.
The wire format is unchanged --- same binary VIRP messages, same HMAC
verification --- over a network socket instead of a Unix socket.

## 5.1 Transport: TCP with TLS 1.3

  -----------------------------------------------------------------------
  **Mechanism**          **What it proves**
  ---------------------- ------------------------------------------------
  HMAC on each message   This observation was signed by the real
                         O-Node\'s key and has not been tampered with

  TLS 1.3 (mutual auth)  This connection is to the real O-Node container,
                         not a man-in-the-middle; channel content is
                         private
  -----------------------------------------------------------------------

## 5.2 Channel Narrowness

The O-Node accepts exactly one kind of request --- execute this command
on this device --- and returns exactly one kind of response --- a signed
VIRP observation. That is the entire surface area of the channel.

Rust\'s type system enforces this at the protocol level. The request
type is a strict schema; the deserializer rejects anything that does not
exactly match before it reaches O-Node logic. Valid actions are an enum
--- only valid actions can be constructed. There is no \'execute
arbitrary code\' action because that variant does not exist in the enum.

## 5.3 Multi-O-Node Federation (Future)

For deployments spanning multiple network segments, a message broker
(NATS) can sit between the agent and multiple O-Nodes. Each O-Node
subscribes to requests for its segment and publishes signed observations
back. Direct TCP between two containers is the correct starting point;
federation is an extension when needed.

## 5.4 Why Not Unix Domain Sockets

The current C O-Node uses a Unix domain socket (/tmp/virp-onode.sock).
UDS is faster than TCP --- no TCP/IP overhead, direct kernel buffer
transfer --- and access control is simply filesystem permissions. For a
single-machine deployment it is a clean, zero-configuration channel.

The fatal constraint is locality. A UDS is a file on the filesystem. Two
containers do not share a filesystem by default, so a UDS cannot span
them. The apparent workaround --- mounting a shared Docker volume
between containers so both can reach the socket file --- directly
undermines the two-container isolation model. Shared filesystem access
is exactly what the IronClaw bypass vectors exploit. Mounting a shared
volume to carry the socket re-opens a subset of those vectors.

  -----------------------------------------------------------------------
                           **Unix Domain Socket vs TCP+TLS**
  ------------------------ ----------------------------------------------
  Transport                Kernel buffer (no network stack) vs TCP/IP
                           stack

  Scope                    Same filesystem only vs across containers,
                           VMs, or machines

  Access control           Filesystem permissions (chmod/chown) vs TLS
                           mutual certificates

  Encryption               None --- relies on OS process isolation vs TLS
                           1.3 confidentiality

  Two-container compatible No --- requires shared volume, which reopens
                           bypass vectors vs Yes

  Federation-ready         No --- single machine only vs Yes --- same
                           channel works across network segments

  Performance              Faster --- microsecond overhead vs Slightly
                           slower --- TCP + TLS handshake on connect,
                           then negligible per-message

  Cert management          None required vs Self-signed private CA for
                           internal use
  -----------------------------------------------------------------------

The performance difference is irrelevant in practice. The O-Node\'s
bottleneck is SSH latency to real devices --- tens to hundreds of
milliseconds per command. TCP+TLS inter-container overhead is
microseconds. The transport choice does not affect observable
performance.

UDS still has a role in the Rust implementation: the virp dev
subcommand, which runs everything in a single process for local
development with the mock driver. In that context there are no
containers, the signing key has no production value, and UDS gives a
fast zero-configuration local loop. It is a development convenience, not
the production architecture.

# 6. O-Node SSH Latency --- The Bottleneck and How to Address It

The O-Node\'s bottleneck is not CPU --- it is I/O wait. SSH command
execution on a real device is mostly waiting: TCP round-trip, SSH
handshake, the device processing the command, the device writing output
back. The O-Node\'s CPU is idle for almost all of that time. The right
solution is concurrency --- doing many things simultaneously while
waiting --- not making any individual operation faster. Tokio\'s async
runtime is already the correct foundation: a small thread pool handles
hundreds of in-flight SSH operations efficiently.

## 6.1 Connection Pooling --- Highest Leverage

The single biggest win. Opening a fresh SSH connection for every
observation request costs 200--500ms before a single byte of device
output is collected: TCP handshake, SSH handshake, key exchange,
authentication. A connection pool keeps authenticated sessions alive and
reuses them. The second request to the same device skips all of that and
goes straight to command execution --- turning a 400ms operation into a
40ms one.

Implementation: a pool per device, keyed by device ID in the O-Node,
holding 1--3 live russh sessions depending on whether the vendor
supports concurrent channels. Idle timeout and health-check reconnection
handle stale sessions. deadpool is a suitable pool crate; a
tokio::sync::Mutex-guarded HashMap works for a simpler first cut.

The critical Cisco IOS complication: interactive shell sessions are
stateful (normal mode, enable mode, config mode). The pool must track
session state and reset it between uses --- returning a session to the
pool in a known clean state. This logic already exists in the C driver
and must be carried faithfully into the Rust pool\'s checkout/return
path.

## 6.2 Parallel Sweep Execution

virp_sweep observes multiple devices simultaneously. In Rust this is
tokio::spawn or futures::join_all over a set of observe futures, bounded
by a semaphore to avoid opening an unbounded number of simultaneous
connections. The 35-router BGP topology completing in under 60 seconds
(current C implementation) is achievable today with serial reconnection.
With connection pooling and bounded parallel execution it should come
down significantly --- most of that 60 seconds is reconnection overhead,
not device response time.

## 6.3 Command Batching for REST API Drivers

For drivers that support it --- FortiGate JSON REST API, Palo Alto XML
API, Juniper NETCONF --- multiple observations can be batched into a
single session or single request. Instead of three sequential
round-trips to collect CPU, interface state, and BGP peers, one NETCONF
get with the right filter returns all three in one round-trip. For
CLI-based drivers like Cisco IOS, commands can be pipelined by sending
them all before reading output rather than send-wait-read sequentially.

## 6.4 Signed Observation Cache

Some observations do not need fresh collection on every request.
Interface descriptions, device hardware inventory, static routing config
--- these change rarely. The O-Node maintains a signed observation cache
with configurable TTLs per command type. A request within the freshness
window returns the existing signed result immediately; stale
observations trigger a fresh collection.

Critical constraint: the cache always returns the original signed
observation --- never a re-signed copy. The HMAC timestamp reflects the
real collection time. The agent and Citation Enforcer see the actual
collection time and evaluate staleness themselves. The O-Node never
backdates or re-signs. This is already consistent with the
observation_freshness and expiry logic in the RFC spec.

## 6.5 What Not to Do --- Pre-fetching

Polling devices constantly to keep fresh data ready generates
unnecessary device load, produces a large mostly-unused observation
corpus, and degrades the freshness guarantee from \'fresh at the moment
you asked\' to \'probably fresh at some recent time.\' For a trust
protocol, on-demand collection with connection pooling is
architecturally cleaner than background polling with speculative
caching.

## 6.6 Implementation Priority

  -----------------------------------------------------------------------
  **Priority**           **Optimization**
  ---------------------- ------------------------------------------------
  1 --- Phase 2,         Connection pooling. Highest return for lowest
  implement from the     complexity. Eliminates the dominant latency
  start                  source.

  2 --- Phase 2, nearly  Parallel sweep execution. Bounded join_all over
  free on tokio          observe futures. Semaphore-limited concurrency.

  3 --- Per-driver as    Command batching for REST API drivers
  each is ported         (FortiGate, Palo Alto, Juniper). Driver-level,
                         not a core change.

  4 --- After core is    Signed observation cache. Useful but adds state
  solid                  management complexity. TTL config per command
                         type.
  -----------------------------------------------------------------------

# 7. New Component: C-Node (Device-Side Attestation)

VIRP cryptographically signs observations at collection time, preventing
fabrication in transit. The remaining frontier is device-side
attestation --- proving the device itself was not lying when queried.
Two complementary approaches close that gap without any vendor
dependency.

## 7.1 Timing Attestation

Real command execution on live hardware has characteristic
nanosecond-level timing signatures that cached or fabricated responses
cannot convincingly replicate. The O-Node captures and cryptographically
binds this timing profile to every observation it signs. A response that
arrives in microseconds from a device that normally takes tens of
milliseconds is flagged as implausible.

## 7.2 C-Node --- Consistency Validator

The C-Node is a separate inspectable process that evaluates whether
observations are physically plausible as a coherent network state. It
operates in three layers:

  -----------------------------------------------------------------------
  **Layer**              **Description**
  ---------------------- ------------------------------------------------
  Hard topological rules Observations that violate physical network laws
                         are rejected immediately (e.g., a BGP session
                         reported up on both sides but with mismatched AS
                         numbers)

  Per-device statistical Each device builds a behavioral baseline from
  models                 its signed observation history. Deviations
                         beyond threshold produce a lowered trust score.

  Whole-topology graph   Cross-device consistency check. An interface
  model                  reported up on Router A must be corroborated by
                         the neighbor on Router B.
  -----------------------------------------------------------------------

The C-Node produces a signed trust score alongside every observation.
Critically, it is trained exclusively on the existing signed observation
corpus --- its reasoning is as auditable as the data it evaluates. It is
not an AI component. It is a deterministic scoring engine over verified
data.

## 7.3 Combined Effect

Together, timing attestation and the C-Node make device-side attestation
a property the infrastructure can earn through consistency rather than
one that requires device vendor cooperation. The cost of a successful
source-level fabrication attack --- making a device lie convincingly in
a way that passes both timing and topological plausibility checks over
time --- becomes dramatically higher.

# 7. Secrets Management Sidecar

Running the O-Node in Docker introduces a trust gap: the container
runtime is a new layer between the O-Node and the hardware. Storing the
O-Key as a plaintext file inside or beside a container means anyone with
host filesystem access can read it. The secrets management sidecar
pattern restores the \'key never touches disk in plaintext\' guarantee
that bare metal deployment provides naturally.

## 7.1 Key Decryption Flow

15. The O-Key is stored encrypted at rest (AES-256) --- never as
    plaintext on disk.

16. On startup, the O-Node container requests decryption from the
    sidecar over an internal Unix socket or Docker internal network.

17. The sidecar authenticates the request and returns the plaintext key
    over the internal channel only --- never exposed externally.

18. The O-Node loads the key into memory. The plaintext key never
    touches disk.

19. virp_key_destroy() / Rust\'s Drop trait handles zeroing the key from
    memory on shutdown --- this is already correct behavior in the
    existing codebase.

## 7.2 Sidecar Authentication Options

  -----------------------------------------------------------------------
  **Level**              **Mechanism**
  ---------------------- ------------------------------------------------
  Simple                 Shared secret between sidecar and O-Node, passed
                         via Docker secrets at startup. Limits exposure
                         but moves the problem one level up.

  Better                 Sidecar checks requesting container identity via
                         Docker API before decrypting. Ensures only the
                         O-Node container can request the key.

  Best (production)      Integrate with HashiCorp Vault, AWS KMS, or
                         Azure Key Vault. Decryption key never exists on
                         the machine --- cloud HSM performs decryption
                         remotely.
  -----------------------------------------------------------------------

## 7.3 The Bootstrap Problem

Something still has to hold the decryption secret. The real answers are:
(1) a human enters a passphrase at startup --- secure but operationally
burdensome; (2) a TPM chip or cloud HSM holds the root secret physically
--- the decryption key cannot be extracted even with full machine
access; (3) remote attestation --- a trusted external service verifies
container integrity before releasing the secret.

The VIRP roadmap already lists \'hardware appliance with TPM-backed
keys\' as a future feature. The existing code is structured to support
it --- virp_key_load_file() and virp_key_destroy() are clean interfaces
a sidecar can slot into without touching core protocol code.

## 7.4 Security Progression

  -----------------------------------------------------------------------
  **Deployment**         **Trust Model**
  ---------------------- ------------------------------------------------
  Bare metal VIRP        Most secure, least convenient. O-Key tied to
                         specific physical machine.

  Docker (no sidecar)    More convenient, weakens trust model. Key on
                         disk, readable by anyone with host filesystem
                         access.

  Docker + Secrets       Restores some trust. Key encrypted at rest; only
  Sidecar                plaintext in memory at runtime.

  Docker + Sidecar +     Closest to bare metal security. Bootstrap
  HSM/TPM                problem solved at hardware level.
  -----------------------------------------------------------------------

## 7.5 Interim Hardening: sodium_mlock()

Device credentials (SSH passwords, API tokens) are currently stored in
plaintext in the virp_device_t struct in memory --- flagged in
virp_driver.h as \'/\* TODO: move to vault/keyring \*/\'. Before full
vault integration, apply sodium_mlock() (or its Rust equivalent via the
secrecy crate) to credential fields in the device struct. libsodium is
already a dependency. This prevents credential fields from being swapped
to disk or appearing in core dumps.

# 8. Rust Crate Dependencies

  -----------------------------------------------------------------------
  **Crate**        **Purpose**               **Why This Crate**
  ---------------- ------------------------- ----------------------------
  russh            SSH client for device     Pure Rust, no
                   drivers                   libssh2/OpenSSL. Supports
                                             legacy ciphers for older
                                             Cisco devices. Async-native
                                             with tokio. Critical for
                                             static binary story.

  rustls           TLS for inter-container   Pure Rust, no OpenSSL. Used
                   channel + HTTPS REST      for TLS on TCP channel and
                   drivers                   Palo Alto REST API
                                             connections. FortiGate no
                                             longer uses HTTPS transport.

  reqwest          HTTP client for REST API  Palo Alto and FortiManager
  (rustls-tls      drivers                   only. FortiGate was rolled
  feature)                                   back to SSH in the v2 C
                                             implementation --- reqwest
                                             is no longer needed for
                                             FortiGate. rustls-tls
                                             feature ensures OpenSSL is
                                             never pulled in.

  hmac + sha2      HMAC-SHA256 signing and   RustCrypto project.
                   verification              Battle-tested. Direct
                                             replacement for OpenSSL HMAC
                                             calls in C implementation.

  hkdf             HKDF-SHA256 session key   RustCrypto project. Used for
                   derivation                per-session key derivation
                                             at SESSION_BIND per draft-03
                                             Section 29. Accepts
                                             transcript_hash as salt and
                                             generation counter as info.

  tokio            Async runtime             Concurrent device
                                             connections, TCP listener,
                                             bounded parallel sweep via
                                             join_all + semaphore.

  deadpool         SSH connection pool per   Manages live russh sessions
                   device                    per device. Eliminates
                                             per-request SSH handshake
                                             overhead (200-500ms).
                                             Handles idle timeout and
                                             health-check reconnection.

  rmcp             MCP server (stdio and SSE Official Rust MCP SDK. May
                   transports)               have rough edges as a
                                             younger crate --- verify
                                             before depending on.

  axum             REST API framework        Minimal, fast, tokio-native.

  serde +          JSON serialization        Request/response layer
  serde_json                                 between agent and O-Node;
                                             MCP tool results.

  tonic + prost    gRPC (optional)           Use if strongly typed
                                             contract between containers
                                             is preferred over raw TCP.
                                             Provides protobuf-defined
                                             schema.

  async-trait      Async trait methods       Required for async methods
                                             in the VirpDriver trait
                                             definition.

  uuid             Observation and proposal  UUID v4 generation.
                   IDs                       

  chrono           Timestamps and freshness  Observation expiry,
                   calculations              staleness checks in the
                                             Intent Broker.

  secrecy          Zeroizing key material    Rust equivalent of
                                             sodium_mlock(). Ensures key
                                             bytes are zeroed on drop.
                                             Replaces virp_key_destroy()
                                             pattern.

  zeroize          Memory zeroing on drop    Used inside secrecy and
                                             independently for any
                                             sensitive buffer.
  -----------------------------------------------------------------------

All of the above are pure Rust with no C system library dependencies.
Building with the musl target produces a fully static binary on Linux.

# 9. What VIRP Guarantees --- and What It Does Not

## 9.1 Structural Guarantees

- If a change was executed, a cryptographically verifiable chain of
  evidence exists showing what observations supported it.

- Those observations were collected by the O-Node from real devices ---
  the AI does not hold the O-Key and cannot produce a valid HMAC without
  it.

- Proposals were submitted with verified, non-stale observation
  references --- the Intent Broker structurally rejects proposals
  without them.

- The appropriate tier of human approval was obtained before execution.

- BLACK tier operations were never transmitted or executed --- they do
  not exist in the wire format or the type system.

## 9.2 The Reasoning Faithfulness Gap

VIRP cannot prevent the AI from reasoning incorrectly or selectively
over real verified data. The four failure modes are:

  -----------------------------------------------------------------------
  **Failure Mode**       **Description**
  ---------------------- ------------------------------------------------
  Selective omission     AI observes R1 has two BGP peers down (verified)
                         but reports \'BGP is largely healthy\' without
                         mentioning them.

  Stale reasoning        Interface was up at 09:00 (verified) and down at
                         14:00 (verified); AI reasons from the earlier
                         observation at 14:30.

  Wrong conclusion from  AI observes a verified OSPF neighbor count drop
  correct data           from 4 to 3 and concludes it is within normal
                         variance when it is not.

  Fabricated causal      AI observes high CPU (verified) and a BGP flap
  chain                  (verified) and invents a causal relationship
                         between them.
  -----------------------------------------------------------------------

This is not a VIRP failure --- it is a fundamental property of language
models and an unsolved problem across the entire field. The Citation
Enforcer (Section 4) is the best available mitigation: deterministic,
not probabilistic, and certain rather than approximate.

## 9.3 The Four-Level Trust Stack

  -----------------------------------------------------------------------
  **Level**              **Status**
  ---------------------- ------------------------------------------------
  L1: Can the AI access  Solved --- VIRP cryptographic observation layer.
  false data?            

  L2: Does the AI        Partially mitigated by Citation Enforcer. Not
  faithfully represent   structurally enforced.
  verified data?         

  L3: Does the AI reason Essentially unsolved. Active research area
  correctly over         across the field.
  faithfully represented 
  data?                  

  L4: Does the AI act on Even less solved. Beyond current VIRP scope.
  correct reasoning in   
  ways matching its      
  intent?                
  -----------------------------------------------------------------------

# 10. Recommended Implementation Order

This sequence minimizes risk by establishing a working, verified
foundation before adding new components.

A note on the future three-role architecture (Section 16): the virp-rs
rewrite targets Stage 1 throughout all phases below --- the single
O-Node model. Nothing in the future architecture changes what Stage 1
builds or how it behaves. The only implication for implementation order
is module discipline inside virp-onode during Phase 2: keep observation
signing logic, intent/policy logic, and write-credential execution logic
in separate modules from the start. In Stage 1 they compile into the
same binary and call each other freely --- no cost, no overhead. The
payoff is that Stage 2 role separation becomes a crate split along an
existing module boundary rather than a refactor that untangles mixed
concerns. The recommended internal layout for virp-onode is:
src/observer/ (signing, HMAC key, read-only device sessions),
src/executor/ (write credentials, command dispatch), src/policy/ (tier
classification, intent evaluation). All three modules live inside
virp-onode for the entire Stage 1 implementation --- only their internal
boundaries matter now.

  -----------------------------------------------------------------------
  **Phase**              **Deliverable**
  ---------------------- ------------------------------------------------
  Phase 1 --- Wire       STOP: before writing any other code, get the
  Format Verification    Rust HMAC output byte-identical to the C
  First                  implementation for the same inputs, verified
                         against the test vectors in VIRP-SPEC-RFC-v2
                         Appendix A. This is the literal first passing
                         test. Build against draft-03 (the current spec),
                         not draft-02. Core types to implement in this
                         phase: VirpObsHeaderV2 (the v2 wire format
                         header with node_id, session_id \[u8;16\],
                         device_id, command_hash \[u8;32\], seq_num),
                         VirpSessionHello/HelloAck/Bind, ONodeAction enum
                         (SESSION_HELLO=20, SESSION_BIND=21,
                         SESSION_CLOSE=22), monotonic generation counter,
                         HKDF-SHA256 session key derivation
                         (transcript_hash as salt, generation as info),
                         and command canonicalization logic that produces
                         command_hash. OKey/RKey as distinct types,
                         TrustTier enum (GREEN/YELLOW/RED only), mock
                         driver, type-level channel separation tests.
                         Session timeouts to enforce: NEGOTIATED state
                         expires after 30 seconds, ACTIVE sessions idle
                         for 5 minutes are reset.

  Phase 2 --- O-Node +   Port the now-complete
  Real Session           HELLO/HELLO_ACK/SESSION_BIND handshake from the
  Handshake + Drivers    C v2 implementation (virp_handshake.c,
                         virp_onode.c). HKDF derivation is formally
                         specified in draft-03 Section 29:
                         transcript_hash =
                         SHA-256(HELLO\|\|HELLO_ACK\|\|SESSION_BIND),
                         session_key = HKDF-SHA256(ikm=master_key,
                         salt=transcript_hash, info=generation). Session
                         key is zeroed on session reset. Generation
                         counter increments on every reset --- stale
                         session IDs cannot be replayed. New HELLO while
                         ACTIVE is rejected with
                         VIRP_ERR_SESSION_INVALID. v2 wire format
                         (draft-03 Section 30) is the default:
                         command_hash = SHA-256(canonical_command),
                         device_id and node_id are bound into every
                         signed observation. Nonce echo check in
                         HELLO_ACK and transcript finalization at
                         SESSION_BIND are load-bearing --- port these
                         carefully from the C implementation. O-Node
                         daemon with TCP/TLS listener. Connection pool
                         (deadpool + russh) from day one. Faithful port
                         of Cisco IOS driver (the hardest case),
                         including stateful session tracking for pool
                         checkout/return. Bounded parallel sweep via
                         join_all + semaphore. Session key caching ---
                         HKDF derivation per connection, not per request.
                         Validate against real hardware before any
                         refactoring. Add remaining drivers in order of
                         production priority. Track every file touched
                         when adding each driver --- device registration
                         currently requires changes across 9+ files in
                         the C implementation. IMPORTANT: Structure
                         virp-onode internals as three separate modules
                         --- observer/, executor/, policy/ --- from the
                         first line of Phase 2 code. They all compile
                         into the same binary in Stage 1; the module
                         separation costs nothing now and prevents an
                         expensive untangling refactor when Stage 2 role
                         separation arrives.

  Phase 3 --- Agent      Intent Broker with R-Key, tier enforcement,
  Container              proposal/approval flow. REST API via axum.
                         Observation store with write queue for sweep
                         burst handling. Freshness tracking. Parallel
                         evidence verification via join_all. Note: the
                         Intent Broker and policy logic in virp-broker
                         maps to the future Policy Node role --- keep
                         tier classification and constraint evaluation
                         clearly separated from observation handling in
                         virp-onode.

  Phase 4 --- MCP Server virp-mcp crate using rmcp. Expose all tools.
                         Decide Citation Enforcer execution model (sync
                         vs. streaming) before this phase --- it affects
                         the tool response schema. Validate end-to-end
                         with Claude Desktop or equivalent frontend.

  Phase 5 --- Citation   virp-audit deterministic rule engine. Structured
  Enforcer               AI output format with citation tags. Display
                         layer rendering verified vs. inferred claims
                         differently.

  Phase 6 --- C-Node +   Timing profile capture and binding in O-Node.
  Timing Attestation     C-Node scoring engine. Per-device statistical
                         models trained on signed observation history.
                         Decide sync vs. async execution model before
                         this phase --- it affects the VirpObservation
                         schema.

  Phase 7 --- Secrets    Encrypted key storage. Sidecar decryption flow.
  Sidecar                sodium_mlock equivalent via secrecy crate on
                         credential fields.

  Phase 8 --- Release    GitHub Actions cross-compilation for Linux
  Pipeline               x86_64, Linux ARM64, macOS Intel, macOS Apple
                         Silicon, Windows. Single-binary release
                         artifacts attached automatically on tag.
  -----------------------------------------------------------------------

# 11. Cargo Workspace Layout

> virp/
>
> Cargo.toml \# workspace
>
> crates/
>
> virp-core/ \# wire format, crypto, types
>
> virp-onode/ \# O-Node daemon + device drivers
>
> virp-broker/ \# Intent Broker, R-Key, tier enforcement
>
> virp-audit/ \# Citation Enforcer
>
> virp-api/ \# REST API (axum)
>
> virp-mcp/ \# MCP server (rmcp)
>
> virp-cnode/ \# C-Node attestation scorer
>
> virp-cli/ \# single binary entry point, subcommands
>
> drivers/
>
> cisco-ios/
>
> cisco-nxos/
>
> cisco-xr/
>
> paloalto/
>
> fortigate/
>
> juniper/
>
> linux/
>
> mock/
>
> tests/
>
> interop/ \# wire format interop with C implementation
>
> integration/ \# end-to-end with mock driver
>
> hardware/ \# optional: real device tests

# 12. Draft-03 Spec Changes --- Wire Format v2 and Session Protocol

VIRP-SPEC-RFC-v2.md advanced from draft-02 to draft-03 in Nathan\'s v2
commit. Three new sections were added that directly define the types and
algorithms virp-core must implement. Build against draft-03 from the
start --- do not port draft-02 types and update later.

## 12.1 Session State Machine (draft-03 Section 28)

The session establishment protocol is now formally specified. The state
machine:

> DISCONNECTED → HELLO_SENT → NEGOTIATED → SESSION_BOUND → ACTIVE →
> CLOSED → DISCONNECTED

Observation and Intent messages MUST NOT be exchanged before
SESSION_BOUND. Required enforcement rules:

  -----------------------------------------------------------------------
  **Rule**               **Detail**
  ---------------------- ------------------------------------------------
  NEGOTIATED timeout     30 seconds --- if SESSION_BIND is not received
                         within 30s of HELLO_ACK, session resets to
                         DISCONNECTED

  ACTIVE idle timeout    5 minutes --- ACTIVE session with no traffic
                         resets to DISCONNECTED

  Socket disconnect      Immediate forced reset to DISCONNECTED on any
                         socket drop

  Single active session  New HELLO while session is ACTIVE is rejected
                         with VIRP_ERR_SESSION_INVALID

  Generation counter     Monotonically increments on every session reset.
                         Stale session IDs from previous sessions cannot
                         be replayed across resets.

  In-memory only         Session state is never persisted. O-Node restart
                         requires a fresh handshake. This is intentional.
  -----------------------------------------------------------------------

## 12.2 Per-Session Key Derivation (draft-03 Section 29)

Following successful SESSION_BIND, the O-Node derives a per-session key
from the master O-Key using HKDF-SHA256. The master key never directly
signs runtime observations. The derivation is precisely specified:

> transcript_hash = SHA-256(serialize(HELLO) \|\| serialize(HELLO_ACK)
> \|\| serialize(SESSION_BIND))
>
> session_key = HKDF-SHA256(ikm=master_observation_key,
> salt=transcript_hash, info=generation)

The generation field is the monotonic counter from the session state
machine --- a uint64 in big-endian encoding passed as the HKDF info
parameter. This ensures a session key derived after a reset is
cryptographically distinct from any previous session key even if the
same master key is used.

  -----------------------------------------------------------------------
  **Property**           **Guarantee**
  ---------------------- ------------------------------------------------
  Master key isolation   The master O-Key never directly signs runtime
                         observations --- only the derived session key
                         does

  Session binding        A session key from a different HELLO exchange
                         will differ even with the same master key,
                         because the transcript hash will differ

  Replay prevention      Stale observations from previous sessions fail
                         verification because the session_id and
                         generation differ

  Forward isolation      Session key is zeroed on session reset ---
                         previous session material cannot be recovered
                         from a compromised current session
  -----------------------------------------------------------------------

Rust implementation: use the hkdf crate (RustCrypto). transcript_hash is
computed with sha2::Sha256 over the concatenated serialized message
bytes. generation is encoded as u64::to_be_bytes() for the info
parameter.

## 12.3 Wire Format v2 --- Context Binding (draft-03 Section 30)

The v2 observation header formally extends what is cryptographically
bound in every signed observation. This closes the attribution ambiguity
present in v1 where a valid payload could theoretically be reassigned to
a different device or command at a higher layer.

  -----------------------------------------------------------------------
  **v1 guarantee**       **v2 guarantee**
  ---------------------- ------------------------------------------------
  Payload authentic ---  Payload + context authentic --- AI cannot
  AI cannot fabricate    fabricate the source, session, device, or
  what it observed       command

  -----------------------------------------------------------------------

The v2 header struct (canonical definition from draft-03):

> version: u8 // VIRP_VERSION_2
>
> channel: u8 // OBSERVATION or INTENT
>
> tier: u8 // GREEN / YELLOW / RED
>
> \_reserved: u8 // must be zero
>
> node_id: u64 // stable O-Node identity
>
> timestamp_ns: u64 // nanoseconds since epoch
>
> seq_num: u64 // monotonically increasing per session
>
> session_id: \[u8; 16\] // from SESSION_BIND
>
> device_id: u64 // stable device identity
>
> command_hash: \[u8; 32\] // SHA-256 of canonical command string
>
> payload_len: u32

A valid v2 observation is a cryptographic commitment to all seven fields
simultaneously: who collected it (node_id), which session (session_id),
which device (device_id), which command (command_hash), what was
returned (payload), when (timestamp_ns), and position in the observation
sequence (seq_num).

## 12.4 Command Canonicalization

command_hash is SHA-256 of the canonical form of the command string.
Canonicalization rules applied in order:

  -----------------------------------------------------------------------
  **Step**               **Rule**
  ---------------------- ------------------------------------------------
  1                      Trim leading and trailing whitespace

  2                      Collapse repeated interior spaces to a single
                         space

  3                      Normalize line endings to LF (\\n)

  4                      Strip CLI prompts and transport-specific
                         wrappers
  -----------------------------------------------------------------------

This must be implemented identically in virp-core and used by every
driver. The canonicalization function is shared --- drivers do not each
implement their own. A command sent over SSH with a trailing prompt
stripped must produce the same hash as the same command sent without a
prompt.

## 12.5 Backward Compatibility

Nodes SHOULD accept both v1 and v2 observations during transition
periods. The negotiated version from HELLO/HELLO_ACK determines which
format is used for new observations. The Rust implementation should
support both during interop testing with the C implementation.

# 13. Open Questions for Pre-Rewrite Decision

  -----------------------------------------------------------------------
  **Question**           **Options / Notes**
  ---------------------- ------------------------------------------------
  Inter-container        Raw TCP + serde_json is simpler to start. gRPC
  protocol: raw TCP vs   (tonic + prost) gives a strongly typed,
  gRPC                   versioned contract. Decide before Phase 2.

  Observation store      SQLite (matches existing chain.db) vs.
  backend                in-memory + WAL. SQLite is the safe choice for
                         continuity with the C implementation\'s
                         tamper-evident chain.

  rmcp maturity          The official Rust MCP SDK is younger than the
                         Python SDK. Evaluate rough edges before
                         committing to it in Phase 4; be prepared to
                         implement a thin JSON-RPC 2.0 server if needed.

  C-Node model storage   Per-device statistical models need a persistence
                         format. SQLite alongside the chain.db is the
                         natural choice. Decide schema before Phase 6.

  C-Node execution       If the C-Node is in the critical path, its
  model: sync vs async   evaluation latency adds to every observation
                         request. The topology graph layer in particular
                         should run asynchronously --- observation
                         returned immediately with \'C-Node evaluation
                         pending\' status, trust score attached when
                         ready. This affects the VirpObservation type
                         schema and MCP tool response format. Decide
                         before Phase 6.

  Citation Enforcer      For long AI responses with many citation tags,
  execution model        synchronous enforcement adds latency to the MCP
                         call. Options: stream results incrementally as
                         citations resolve, or return a preliminary
                         response with an async follow-up audit report.
                         Affects the MCP tool response schema. Decide
                         before Phase 4.

  Sidecar language       The sidecar can be Go or Python. A Rust
                         implementation keeps the codebase uniform.
                         Decide before Phase 7.

  Wire format backward   The Rust rewrite should produce byte-identical
  compatibility          HMAC values for the same inputs to maintain
                         interop with the C and Go implementations. Test
                         vectors from VIRP-SPEC-RFC-v2 Appendix A are the
                         acceptance criteria.
  -----------------------------------------------------------------------

# 14. Additional Bottlenecks and Architectural Notes

Beyond SSH latency (Section 6), several other bottlenecks and design
decisions are worth addressing before or during implementation. Ordered
by likelihood of causing production problems.

## 13.1 Observation Store Write Contention Under Sweep Load

When virp_sweep fires across 35 devices in parallel, all observations
complete around the same time and all attempt to write to the SQLite
store simultaneously. SQLite\'s WAL mode handles concurrent reads well
but serializes writes. A full-topology sweep produces a burst of \~35
write contentions within the same 100ms window --- this will not show up
in mock-based testing and only surfaces on real sweeps.

The fix is a short write queue that absorbs the burst and batches it
into a single transaction. Design the observation store interface with
this in mind from Phase 3, not as a retrofit. The queue sits between the
O-Node\'s signing output and the SQLite write path --- observations are
acknowledged to the agent immediately after signing; persistence is
slightly deferred and batched.

## 13.2 Intent Broker Sequential Verification Loop

When the AI submits a proposal citing many observations as evidence, the
current design verifies them one at a time: retrieve, check verified
flag, check freshness, repeat. For a well-evidenced change proposal this
is a serial loop over potentially many store reads.

This is straightforwardly parallelizable with join_all over the
verification futures --- the same pattern as the sweep. There is also a
correctness argument: if any cited observation fails verification the
whole proposal is rejected, so discovering all failures simultaneously
is better than stopping at the first and making the AI submit revised
proposals iteratively. Implement parallel verification from the start in
Phase 3.

## 13.3 TLS Handshake Cost on the Inter-Container Channel

TLS 1.3 reduced the handshake to one round-trip, but if the agent opens
a new TLS connection to the O-Node for every observation request, that
cost is paid on every call. Under a 35-device sweep that is 35 TLS
handshakes in parallel --- not catastrophic, but unnecessary.

Use a persistent TLS connection with connection reuse, or a small pool
of pre-established connections between the agent and O-Node. This is
cheap to implement correctly from the start and disruptive to retrofit
later since it affects the connection management layer shared by all
request types.

## 13.4 C-Node as a Synchronous Gate

If the C-Node is in the critical path --- the O-Node waits for its trust
score before returning the observation --- then C-Node evaluation
latency is added to every observation request. The hard topological
rules layer is fast. The per-device statistical model layer requires a
store read per device. The whole-topology graph layer potentially
requires reading state for many devices.

The topology graph layer in particular must run asynchronously. The
observation is returned immediately with a \'C-Node evaluation pending\'
status; the trust score is attached when the C-Node finishes. The agent
and Citation Enforcer treat a missing score as \'not yet evaluated\'
rather than blocking. This is an architectural decision that affects the
VirpObservation type schema and the MCP tool response format --- it must
be settled before Phase 6, not discovered during it.

## 13.5 Key Derivation at Session Establishment

The C implementation uses HKDF to derive session keys from the master
O-Key --- the master key never signs runtime observations directly,
which is correct. If session establishment involves HKDF derivation on
every new agent connection (restarts, health checks, network blips), key
derivation work happens more often than necessary.

Session keys should be cached for the lifetime of the TLS connection and
only re-derived when the connection is genuinely new. Make this explicit
in the session management design so it does not accidentally become
per-request in the Rust implementation.

## 13.6 Citation Enforcer in the Synchronous MCP Response Path

The Citation Enforcer receives the AI\'s full response text and must
parse it for citation tags, look up each cited observation ID in the
store, and produce an audit report --- all before the MCP tool call
returns to the frontend. For short responses with a few citations this
is fine. For a complex multi-device analysis with many citation tags it
adds noticeable latency to what the user experiences as the AI\'s
response time.

Options: stream enforcer results back incrementally as each citation is
resolved, or return the AI\'s response immediately and deliver the audit
report as a follow-up event. Either approach requires a deliberate
decision about the MCP tool response schema before Phase 4. Retrofitting
streaming into a schema designed for synchronous responses is a breaking
change.

## 13.7 What Is Not a Bottleneck

HMAC-SHA256 computation is nanoseconds per observation --- the signing
step will never appear in any real profile. Serde JSON serialization and
deserialization for inter-container messages is similarly fast. All
meaningful bottlenecks are I/O: SSH to devices, store writes, TLS
connections, and any point where a synchronous operation is accidentally
placed in an async path. When in doubt, profile against a real 35-device
sweep before optimizing --- the numbers from mock testing will not
predict production behavior.

## 13.8 Priority Summary

  -----------------------------------------------------------------------
  **Priority**           **Item**
  ---------------------- ------------------------------------------------
  1 --- SSH connection   Dominant latency source. Covered in Section 6.
  pool (Phase 2, day     
  one)                   

  2 --- Observation      Burst write contention under sweep. Cheap to
  store write queue      design in, disruptive to retrofit.
  (Phase 3, design-time) 

  3 --- TLS connection   Persistent connection between agent and O-Node.
  reuse (Phase 2, day    Cheap and structural.
  one)                   

  4 --- Intent Broker    join_all over evidence verification. Correctness
  parallel verification  improvement that also happens to be faster.
  (Phase 3)              

  5 --- C-Node async     Architectural decision affecting VirpObservation
  execution model        schema and MCP response format.
  (decide before Phase   
  6)                     

  6 --- Citation         Sync vs. async/streaming. Affects MCP tool
  Enforcer execution     response schema.
  model (decide before   
  Phase 4)               

  7 --- Session key      HKDF derivation should not be per-request.
  caching (Phase 2)      Explicit in session design.
  -----------------------------------------------------------------------

# 15. Development Practices --- Working with Claude Code

This section documents a class of failure specific to AI-assisted
development that was observed in production during IronClaw. It is not a
theoretical concern.

## 14.1 The Silent Bypass Problem

During IronClaw development, a significant generation session produced
an integration that looked complete on surface review --- the code
described the right architecture, the structure was plausible, tests
passed. What had actually happened: the Python layer had quietly
reimplemented the C routing logic in Python. The ctypes bindings
referenced symbols that did not exist. The FortiGate path never touched
the C implementation at all. None of this was visible from reading
descriptions of the code or from surface-level review.

The failure mode is specific to AI code generation: the generated code
describes the correct architecture in its structure and naming, but the
actual execution path diverges silently from what the architecture
requires. The code looks right. It does not do what it looks like it
does.

## 14.2 The Standing Practice: Trace One Complete Path After Every Significant Session

After any significant Claude Code generation session, before considering
the work done, trace one complete execution path end-to-end in the
actual code --- not in the description of the code, not in the test
output, in the source itself. Follow a real request from entry point
through every function call to the final signed output. Verify that each
step in the path is what the architecture requires it to be.

For the VIRP rewrite, the canonical path to trace after each session is:

20. MCP tool call received by virp-mcp

21. Request serialized and sent over TLS TCP to O-Node

22. O-Node deserializer validates schema --- confirm it rejects on
    mismatch

23. Request routed to correct vendor driver --- confirm the driver is
    the Rust implementation, not a reimplementation or stub

24. Driver executes command via russh SSH session from the connection
    pool

25. Raw output returned to O-Node core

26. O-Node signs output with O-Key via HMAC-SHA256 --- confirm key is
    the real O-Key, not a test key or placeholder

27. Signed VirpObservation returned to agent over TLS TCP

28. Observation written to store and returned to MCP caller

If any step in this trace is not what the architecture specifies --- if
a function routes somewhere unexpected, if a binding references a symbol
that does not exist, if a signing step uses a placeholder --- that is a
generation error that surface review missed. Fix it before the next
session builds on top of it.

## 14.3 Specific Things to Check in the VIRP Context

  -----------------------------------------------------------------------
  **Risk**               **What to verify in the trace**
  ---------------------- ------------------------------------------------
  Driver bypass          The vendor driver path actually calls russh and
                         reaches a real device. No Python
                         reimplementation, no stub returning mock data on
                         the real driver path.

  Key substitution       The O-Key used for signing is loaded from the
                         configured key source, not a hardcoded test
                         value that survived from a mock.

  Channel confusion      Signing functions are called with the correct
                         key type. O-Key signs observations; R-Key signs
                         proposals. No cross-channel calls.

  Session_id placeholder Once Phase 2 implements the real handshake,
                         verify session_id is populated from the
                         handshake, not still zeroed from the Phase 1
                         placeholder.

  Schema enforcement     The O-Node deserializer actually rejects a
                         malformed request. Send one deliberately and
                         confirm rejection before trusting the
                         enforcement.

  Store writes           Observations are reaching SQLite, not being
                         silently dropped. Check the actual row count
                         after a sweep, not just the response objects.
  -----------------------------------------------------------------------

# 16. Future Architecture --- Role Separation (draft-howard-virp-02 Section 9)

Nathan added Section 9 to the paper documenting a planned evolution of
the O-Node architecture. This is not a current implementation
requirement --- the virp-rs rewrite targets the existing single O-Node
model (Stage 1 below). It is documented here because the crate
boundaries chosen now should not make this evolution harder later.

## 16.1 The Problem with a Single O-Node

The current O-Node is a single process that concentrates three distinct
capabilities: collecting and signing observations (holding the HMAC
chain key), evaluating intents against policy (holding the constraint
ledger and approval authority), and executing write operations on
devices (holding write credentials). A compromised O-Node gives an
attacker all three simultaneously --- the ability to fabricate state,
authorize actions, and carry them out.

## 16.2 The Three-Role Architecture

The future architecture decomposes the O-Node into three principals,
each holding exactly one of the three capabilities and nothing else:

  -----------------------------------------------------------------------
  **Role**         **What it holds**         **What it cannot do**
  ---------------- ------------------------- ----------------------------
  Observer Node    HMAC chain key +          Approve intents or execute
                   read-only device          write operations on devices
                   credentials               

  Executor Node    Write credentials for     Generate signed command
                   managed devices           bundles without a valid
                                             Policy Node signature ---
                                             holds no chain key, no
                                             policy state

  Policy Node      Constraint ledger, trust  Touch devices directly or
                   tier definitions,         hold device credentials of
                   approval queue, Ed25519   any kind
                   signing key for command   
                   bundles                   
  -----------------------------------------------------------------------

The separation-of-duties property this enforces: no single node can
observe device state, approve an action, and execute that action. Each
capability requires a different principal with independent keying
material.

## 16.3 Blast Radius Analysis

The security value of the three-role architecture is understood most
clearly by analyzing what an attacker gains from compromising each node
individually and in combination:

  -----------------------------------------------------------------------
  **Compromise           **Blast radius**
  scenario**             
  ---------------------- ------------------------------------------------
  Observer Node alone    Can forge observations and read device state
                         passively. Cannot approve intents (no Policy
                         Node key). Cannot execute writes (no write
                         credentials).

  Executor Node alone    Has write credentials but cannot generate
                         validly signed command bundles without the
                         Policy Node key. Bounded to replaying previously
                         approved commands if replay protections are
                         absent.

  Policy Node alone      Can sign malicious command bundles but has no
                         path to devices and no execution capability.
                         Inert unless Executor is also compromised.

  Observer + Executor    Can forge state and holds write credentials but
                         still cannot generate new signed command
                         bundles. Policy Node audit trail remains clean
                         and shows no corresponding approval for
                         unauthorized actions.

  Policy + Executor      Most dangerous two-node scenario: can authorize
                         and execute arbitrary configuration changes.
                         Cannot forge observations --- the uncompromised
                         Observer Node faithfully records what actually
                         happened, providing a forensic trail of the
                         attack\'s impact.

  Observer + Policy      Can forge observations and pre-sign malicious
                         bundles but has no execution capability. Bundles
                         remain inert without Executor compromise or
                         separate credential theft.

  All three nodes        Equivalent to the current single O-Node
                         compromise --- full observation forgery,
                         arbitrary approval, unrestricted device writes.
                         The architecture does not defend against
                         simultaneous three-node compromise. Its value is
                         requiring three independent breaches instead of
                         one.
  -----------------------------------------------------------------------

## 16.4 Migration Roadmap

Nathan defined four incremental stages, each independently deployable
and each providing immediate security benefit:

  -----------------------------------------------------------------------
  **Stage**              **Description**
  ---------------------- ------------------------------------------------
  Stage 1 --- Single     Existing single-process O-Node handles
  O-Node (current)       observation, policy, and execution. All VIRP
                         cryptographic guarantees and all seven trust
                         primitives are enforced. This is the virp-rs
                         Phase 1--8 implementation target. Blast radius
                         of O-Node compromise is the full device set.

  Stage 2 ---            O-Node splits into Observer process (HMAC key +
  Observer-Executor      read-only credentials) and Executor process
  separation             (write credentials + signature verification).
                         Policy evaluation remains co-located with one
                         process. Achievable on a single host using
                         process isolation, separate users, or
                         containers.

  Stage 3 --- Policy     Policy evaluation, constraint ledger, and
  Node extraction        approval queue extracted into a dedicated Policy
                         Node with its own Ed25519 signing key. Executor
                         modified to require a valid Policy Node
                         signature on all command bundles. All three
                         roles now operate as separate processes.

  Stage 4 --- Physical   Observer, Executor, and Policy Nodes deployed on
  separation             separate hosts with independent network
                         policies. Policy Node optionally air-gapped or
                         HSM-backed. Inter-node communication
                         authenticated via mutual TLS.
  -----------------------------------------------------------------------

## 16.5 Implications for virp-rs Crate Design

The virp-rs rewrite targets Stage 1. However, the Cargo workspace crate
boundaries should not require a major restructuring to reach Stages 2
and 3. The existing layout already maps well:

  -----------------------------------------------------------------------
  **Future role**        **Current virp-rs crate**
  ---------------------- ------------------------------------------------
  Observer Node          virp-onode --- observation collection, signing,
                         driver execution. Already the natural home for
                         the HMAC chain key and read-only device
                         sessions.

  Policy Node            virp-broker + virp-audit --- Intent Broker holds
                         R-Key and policy state; Citation Enforcer holds
                         constraint logic. These already form a natural
                         policy boundary.

  Executor Node          No dedicated crate yet --- currently execution
                         is part of virp-onode. Stage 2 separation would
                         extract write-credential execution into a
                         dedicated virp-executor crate that accepts only
                         signed command bundles.
  -----------------------------------------------------------------------

The practical guidance: do not conflate observation signing logic with
intent execution logic inside virp-onode. Keep them in clearly separated
modules from the start. Stage 2 separation then becomes a crate split
along an existing module boundary rather than a refactor that untangles
mixed concerns.

## 16.6 Security Considerations

Three specific risks introduced by role separation that Stage 1 does not
have:

  -----------------------------------------------------------------------
  **Risk**               **Mitigation**
  ---------------------- ------------------------------------------------
  Inter-node             Every new communication path between roles is an
  communication paths    attack surface. Each MUST be protected by
                         authenticated and encrypted transport. Network
                         topology alone is not sufficient authentication.

  Key management         Stage 1 has one HMAC key and one Ed25519 key
  complexity             pair. Three-role architecture adds a second
                         Ed25519 key pair for the Policy Node and
                         requires secure public key distribution to
                         verifying nodes. Key rotation procedures must be
                         defined per role independently.

  Command bundle replay  The Executor must reject bundles whose sequence
  at Executor Node       number has already been processed or whose
                         timestamp exceeds a staleness threshold.
                         Implement monotonic sequence numbers and
                         expiration timestamps in signed command bundles
                         from Stage 2 onward.
  -----------------------------------------------------------------------

------------------------------------------------------------------------

*VIRP --- Verified Infrastructure Response Protocol \| Apache 2.0*
