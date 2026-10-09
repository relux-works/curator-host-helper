# curator-host-helper: specification

- **Status:** draft v0.2 (2026-10-09). Nothing is implemented.
- **Normative language:** MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.
- **Companions:** [curator-credential-broker](https://github.com/relux-works/curator-credential-broker) (binds the OS accounts this helper creates), [curator-network-profiles](https://github.com/relux-works/curator-network-profiles) (the egress an agent may use).

### Revision 0.2

An architecture review on 2026-10-09 kept the closed, root-owned design and asked for tighter contracts. This revision adds a second binary, the **launcher**, which starts the executor under an agent's account (§7); without it an unprivileged dispatcher could create an account but never run anything as it. It also adds per-caller policy (§3), an account lifecycle with one lock and a retirement fence (§4, §5), descriptor-relative file operations (§6), an exact sudo and input contract (§2), and a firewall support matrix with trusted applied state (§8).

## 1. Purpose

An agent system gets a real boundary between agents on one machine by running each agent under its own OS account: processes of different accounts cannot read each other's files or memory, and kernel firewalls can match traffic by account. Creating accounts, starting processes under them and installing firewall rules need root. This repository holds the only components with that power, and each holds it narrowly:

- **curator-host-helper** creates, lists and retires accounts and (v1) applies per-account firewall rules;
- **curator-host-launcher** starts one approved executable under one active agent account.

Both run as root for one operation at a time, are invoked through `sudo` by configured callers, accept only operations from closed, versioned schemas, choose names and identifiers themselves, act only on what they manage, journal their steps, and serialise on one lock.

### 1.1 Non-goals

- They do not decide who may start an agent (the dispatcher does) and do not hold credentials (the broker does).
- The helper never starts processes; the launcher never creates accounts or rules.
- Neither accepts free-form commands, paths, shell fragments or environment.

## 2. Invocation

- Each binary is installed at a fixed root-owned path, is not setuid, and every directory on its path is root-owned and not writable by others.
- The sudoers rule allows a configured caller to run exactly that path **with no arguments**, using the empty argument specification, for example `cur-dispatch ALL=(root) NOPASSWD: /usr/local/libexec/curator-host-helper ""`. The binaries also refuse any argument (`argv_not_allowed`).
- The binary refuses unless its effective UID is 0 (`not_root`) and `SUDO_UID` is present and a plain decimal number (`not_invoked_by_sudo`). `SUDO_UID` is the caller's identity only because sudo set it during a trusted transition; it is not proof of the caller's executable, and root callers are outside this boundary.
- The binary clears its environment, sets a fixed `PATH`, working directory and umask, and calls operating-system tools only by absolute path, never through a shell.
- The request is exactly one JSON document on stdin, at most 16 KiB, read within 5 seconds (`request_too_large`, `request_timeout`). Duplicate keys, unknown members, a trailing document, strings longer than their declared bound and arrays longer than theirs are refused (`request_invalid`).
- The answer is one JSON document on stdout; diagnostics go to the audit log.

```
helper-op/1
{ v: 1,
  op:         "user.create" | "user.retire" | "user.list" | "fw.apply" | "fw.remove",
  request_id: string (≤ 64),           // idempotency key, §4.4
  args:       { … per operation … } }

helper-result/1
{ request_id, ok: true, result: { … }, state: current state of the target }
| { request_id, ok: false, error: { code, message } }
```

A `grants` member is refused until v2 (`grants_unsupported`), so an ignored grant can never look verified.

## 3. Policy

A root-owned configuration lists callers by UID and gives each a role:

| Role | May |
|---|---|
| `installer` | create and retire **service** accounts it names in the configuration |
| `dispatcher` | create agent accounts up to its quota; retire, launch and apply rules only for agent accounts **it created** (`created_by` = its principal) |
| `operator` | everything above, including retiring protected service accounts |

Requests outside the caller's role are refused (`caller_role_not_permitted`, `target_not_owned`, `quota_exceeded`). Configured dispatchers are trusted administrators of their own agents; until v2 (§9) the policy is the authority.

## 4. Accounts

### 4.1 Names and identifiers

- Agent accounts are named `cur-a-<label>`, service accounts `cur-s-<name>`. `label` and `name` MUST match `[a-z0-9][a-z0-9-]{0,22}` (`label_invalid`).
- UIDs and GIDs come from configured ranges, one for agents and one for services, chosen per OS so that the accounts stay hidden from login windows and do not collide with system accounts. The helper picks the lowest value that is free both in the directory service and in its ledger, under the helper lock (§4.4); it never accepts a UID from the caller.
- Each account gets its own primary group with the same name and number and no supplementary groups.
- Each creation gets a **generation**: an identifier that is never reused, even when the UID or label is.

### 4.2 Lifecycle

```
creating → active → retiring → removed
    └─ (failure) → rolled back
```

- `creating`: steps are journaled; a failure or crash rolls the creation back.
- `active`: the account exists and may be launched (§7) and bound by the broker.
- `retiring`: the commit point of removal. From here the launcher refuses the generation, the broker refuses its leases, and removal only moves forward, resuming after a crash.
- `removed`: the generation stays in the ledger forever.

### 4.3 `user.create`

```
args:   { kind: "agent" | "service", label: "dev-7f3" }
result: { username: "cur-a-dev-7f3", uid: 612, gid: 612, home: "<agents root>/cur-a-dev-7f3",
          generation: "g-0193", state: "active" }
```

The account has no usable password, no login shell (`/usr/bin/false` on macOS, `/usr/sbin/nologin` on Linux) and no supplementary groups. The home is created empty under a fixed root (agents and services have separate roots), owned by the account, mode 0700, through descriptors (§6). The helper refuses when the name or home already exists outside its ledger (`user_exists_unmanaged`, `home_exists_unmanaged`).

- **macOS:** records are created with `dscl` in the local node (`UniqueID`, `PrimaryGroupID`, `UserShell`, `NFSHomeDirectory`, `IsHidden`, a disabled password), not with `sysadminctl`, so that no FileVault or secure-token prompt can occur; such accounts can never unlock the disk or log in, which is intended.
- **Linux:** `groupadd` and `useradd` with explicit `--gid`, `--uid`, `--home-dir`, `--no-create-home`, `--shell /usr/sbin/nologin` and a locked password, with distribution defaults that would create extra state disabled; the home is created by the helper.

Qualification verifies the result on each OS: authentication disabled, no elevated or supplementary groups, exact UID, GID and home fields, and that a service can run under the account without a graphical login.

### 4.4 `user.retire`

```
args:   { generation: "g-0193", home: "archive" | "delete" }
result: { username, generation, state: "removed", archive?: path }
```

1. Persist `retiring` (commit point). The launcher refuses new starts for the generation from now on.
2. Ask the broker to end the generation's binding where one exists (the dispatcher normally unbinds first; the broker also refuses any retiring generation on its own, broker §4.2).
3. Stop the account's per-user service domain where the OS has one, terminate every process of the UID and verify that none remain for a quiet period (`processes_remain` leaves the account in `retiring` for a later retry).
4. (v1) Keep the account's egress denial in place until step 6.
5. Move the home into a root-owned quarantine directory on the same file system, then archive or delete it with a descriptor-relative walk (§6).
6. Remove the account and group, then the firewall rules, and persist `removed`.

### 4.5 `user.list` and the ledger

The ledger is a root-owned, world-readable file with one record per generation: `{ generation, username, uid, gid, kind, label, state, created_at, created_by, retired_at? }`. It contains no secrets. It is replaced atomically (temporary file, flush, rename, directory flush) in a root-owned directory. Other components (the broker, the launcher) read it directly; `user.list` returns the same records. Readability is not authority.

### 4.6 Lock, journal and idempotency

- One durable lock (an exclusive lock on a root-owned file) serialises every operation of both binaries, including recovery and allocation.
- Every operation writes generation-scoped journal records of planned and completed steps before acting. Recovery runs first in every invocation and compares the actual operating-system objects with the journal before each action: an interrupted creation is rolled back, an interrupted retirement is resumed.
- `request_id` is scoped to the caller's principal. A repeated `(caller, request_id)` with the same operation and canonical arguments returns the recorded result together with the target's **current** state, so an old success never looks like a live account; different arguments are refused (`request_id_conflict`).
- Every operation appends one line to a root-owned audit log: time, caller, operation, arguments, result.

## 5. Cross-component reconciliation

A dispatcher's request id ties together the helper's `user.create`, the broker's `bind` and the launcher's `launch.start`. After a failure between them, the dispatcher repeats the same request ids: each component returns its recorded result, and the dispatcher completes the missing step or retires the generation.

## 6. File operations

- Control paths (configuration, ledger, journal, audit, quarantine, archive, the homes' roots) are root-owned directories that are not writable by others; a control path that is a symbolic link refuses (`path_is_link`).
- All operations work relative to trusted directory descriptors and never follow links: on Linux through `openat2` with `RESOLVE_BENEATH` and `RESOLVE_NO_SYMLINKS`; on macOS through a per-component `openat` walk with `O_NOFOLLOW` (`O_NOFOLLOW_ANY` where available). Every opened object is checked with `fstat` for type, owner and device where it matters.
- Homes are created with `mkdirat`, `fchown` and `fchmod` on the descriptor. The helper never runs a recursive `chown` over content an agent controls.
- Inside a home, symbolic links are ordinary objects: archiving records them as links and deletion unlinks them; neither follows them. A home therefore never becomes unremovable because it contains links.
- Destructive walks start only after the account is quiescent (§4.4) and the home has been moved into quarantine, so no process of the account can swap a directory during the walk.

## 7. The launcher (`curator-host-launcher`)

The launcher is the narrow transition from a dispatcher into an agent account that the platform design calls for: it starts one approved executor, and nothing else, under one active agent generation.

```
launch-op/1
{ v: 1,
  op:         "launch.start",
  request_id: string (≤ 64),
  generation: "g-0193",
  executor:   "curator-executor",          // an id from the approved list
  cwd:        "work/run-42",               // relative to the agent's home
  plan:       base64 (≤ 64 KiB) }          // delivered to the executor on descriptor 3
```

1. Under the shared lock, read the ledger: the generation is `active`, of kind agent, and created by the calling dispatcher (`generation_not_active`, `target_not_owned`).
2. Resolve `executor` in the root-owned approved list to an absolute path in a root-owned directory and a SHA-256 digest; open the file without following links and verify the digest (`executor_not_approved`, `executable_digest_mismatch`).
3. Open `cwd` beneath the agent's home without following links (`cwd_invalid`).
4. Close every descriptor above 2, then create a pipe, write `plan` into it and place its read end at descriptor 3. Standard input, output and error are the caller's (its pipes or terminal); no other descriptor crosses the transition.
5. Build the environment from a fixed set (`HOME`, `USER`, `LOGNAME`, `PATH`, `LANG`, `TMPDIR` inside the home); nothing is inherited.
6. Set the supplementary groups to none, then the GID, then the UID (real, effective and saved), and verify that root cannot be regained (`privilege_drop_failed`).
7. Exec the verified executable (on Linux by descriptor with `execveat`; on macOS by its path in the root-owned directory).

sudo stays the parent of the executor and relays signals, so the dispatcher owns the process's lifetime through the sudo process it started. The executor is trusted platform code: it reads the plan, resolves the protected credential binding, obtains its lease from the broker on a fresh connection and execs the harness (broker §12.2). Arbitrary `sudo -u` or a general command runner is never a substitute for the launcher.

## 8. Firewall per account (v1)

```
fw.apply  args: { generation, profile_ref, profile_digest, proxy: { port } }
fw.remove args: { generation }
```

- **Intent.** An agent account reaches the network only through its network profile's local proxy; other traffic of the account is denied. Unix-domain sockets (the broker, the message bridge) are not filtered.
- **macOS:** one `pf` anchor per account under a helper-owned parent anchor using `user <uid>` rules; the parent anchor's position in the main ruleset and pf's enabled state are verified on every apply. **Linux:** one helper-owned nftables table with `meta skuid <uid>` rules.
- **Support matrix.** pf associates only TCP and UDP traffic with an account. v1 states, per OS release, which of these it enforces with evidence: TCP and UDP over IPv4 and IPv6, loopback and external destinations, DNS and QUIC, sockets created before the rules, and the interaction with system rules and existing states. Anything an unprivileged account can do outside that matrix (other protocols, for example) needs an additional restriction applied by the launcher (such as a sandbox profile that limits socket creation) or makes the tuple unsupported (`fw_unsupported`). No profile is reported as enforced until its whole matrix has evidence.
- **Order.** For an account whose profile requires enforcement, denial is installed in the same locked operation that activates the account, before the first launch; it is removed only at the end of retirement (§4.4). Rule changes for a running account flush that account's existing states.
- **Proxy.** The proxy runs under a trusted service account with a defined listener lifecycle. `fw.apply` verifies that the listener on the port belongs to that service account; the allow list holds only that destination.
- **Applied state.** After a successful apply the helper publishes a root-owned, world-readable record per generation: `{ generation, uid, profile_ref, profile_digest, proxy: { service generation, port }, firewall_generation, applied_at }`. The broker matches leases against this record (broker §5.5), never against a claim from the agent's own process.

## 9. Grant verification (v2)

In the platform design the helper is a handler process that independently limits authority. v2 checks, for every request, a grant chain to a configured root that authorises the action on the concrete target in the current time window and is not revoked, and verifies that the caller is the grant's subject (for example by a signature of the subject key over the request). A valid chain presented by someone else is not authority. The role policy of §3 remains as defence in depth.

## 10. How it fits

| Caller | Operations | Then |
|---|---|---|
| dispatcher (v0: `curator agent-user`) | `user.create {agent}`, `user.retire`, `fw.apply`, `fw.remove`, `launch.start` | binds the generation in the broker before `launch.start`; unbinds before `user.retire` |
| platform installer | `user.create {service}`, `user.retire {service}` | installs the broker and other services under those accounts |
| broker | reads the ledger and the applied state | checks generations and network state before leasing |

## 11. Installation

The installer places both binaries, the root-owned configuration (callers and roles, quotas, UID and GID ranges, home roots, quarantine and archive directories, the approved executor list with digests, proxy service accounts), the ledger, journal and audit directories, the two sudoers rules and, for v1, the parent firewall anchor. Uninstall refuses while managed accounts exist unless an operator forces it.

## 12. Qualification

Tests that create accounts, start processes under them or change firewall rules run only on hosted CI runners (macOS and Ubuntu runners allow `sudo`), never on a developer's or a production host:

- create, list, retire, idempotent repeat, `request_id` conflict, current state on a repeated old request;
- policy: a dispatcher cannot retire, launch or change rules for another dispatcher's agent or for a service account; quotas;
- invocation: arguments refused (also through the sudoers rule), non-root and missing or malformed `SUDO_UID` refused, oversized, duplicate-key and trailing-document requests refused, a `grants` member refused;
- concurrency: two creations racing for the same UID; a creation racing with a retirement;
- crash recovery after every step: creation rolled back, retirement resumed;
- files: a rename race during a walk, a control path replaced by a link, a home that contains links is archived and deleted without following them;
- launcher: unapproved executor, digest mismatch, retiring generation, another dispatcher's generation, `cwd` escaping the home, no descriptor above 3 reaches the executor, supplementary groups cleared, root cannot be regained;
- v1: the support matrix for each OS release, including denied and allowed destinations, other accounts unaffected, rules present before the first launch and kept until retirement ends, applied state published and removed.

## 13. Versions

| Version | Content |
|---|---|
| v0 | accounts (create, retire, list), policy, ledger, lock, journal, audit, the launcher; macOS and Linux |
| v1 | per-account firewall rules with the support matrix and applied state |
| v2 | grant verification; parent read access to a child's home (ACL); sandbox profiles for donor machines |

## 14. Open questions

1. Home roots, and whether a parent orchestrator gets read access to its children's homes (ACL) in v0 or v2.
2. Archive or delete homes by default on retirement.
3. UID and GID ranges per OS.
4. The restriction for traffic outside pf's TCP and UDP matching on macOS (§8).
5. Windows (later).

## Appendix A. Error codes

`argv_not_allowed`, `not_root`, `not_invoked_by_sudo`, `caller_not_permitted`, `caller_role_not_permitted`, `target_not_owned`, `quota_exceeded`, `request_too_large`, `request_timeout`, `request_invalid`, `op_unknown`, `request_id_conflict`, `grants_unsupported`, `label_invalid`, `range_exhausted`, `user_exists_unmanaged`, `home_exists_unmanaged`, `generation_not_active`, `state_conflict`, `processes_remain`, `path_is_link`, `step_failed_rolled_back`, `executor_not_approved`, `executable_digest_mismatch`, `cwd_invalid`, `privilege_drop_failed`, `fw_target_not_allowed`, `fw_unsupported`.

## Appendix B. Diagrams

PlantUML sources in [`diagrams/`](../diagrams/): `account-lifecycle.puml`, `launch.puml`.
