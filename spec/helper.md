# curator-host-helper: specification

- **Status:** draft v0.1 (2026-10-09). Nothing is implemented.
- **Normative language:** MUST, MUST NOT, SHOULD and MAY are used as in RFC 2119.
- **Companions:** [curator-credential-broker](https://github.com/relux-works/curator-credential-broker) (binds the OS users this helper creates), [curator-network-profiles](https://github.com/relux-works/curator-network-profiles) (the egress an agent may use).

## 1. Purpose

An agent system gets a real boundary between agents on one machine by running each agent as its own OS user: processes of different users cannot read each other's files or memory, and kernel firewalls can match traffic by user. Creating users and firewall rules needs root. This helper is the only component that holds that power, and it holds it narrowly:

- it runs as root for one operation at a time, invoked through `sudo` by an allowed caller;
- it performs only operations from a closed, versioned schema;
- it chooses names and identifiers itself, removes only what it created, journals every step, and rolls back on failure.

### 1.1 Non-goals

- It does not decide who may start an agent (the dispatcher does) and does not hold credentials (the broker does).
- It does not run agents, manage homes' content or install software.
- It never accepts free-form commands, paths or shell fragments.

## 2. Invocation

- The helper is a single static binary installed at a fixed root-owned path; its directory and every parent are root-owned and not writable by others.
- A sudoers rule allows the configured callers to run exactly that absolute path with no arguments, for example `cur-dispatch ALL=(root) NOPASSWD: /usr/local/libexec/curator-host-helper`.
- The request is one JSON document `helper-op/1` on stdin (at most 16 KiB); the answer is one JSON document `helper-result/1` on stdout; diagnostics go to the audit log, not to stdout.
- The caller is the UID in `SUDO_UID`, which sudo sets. The helper MUST refuse to run when invoked without sudo, or when `SUDO_UID` is not in the root-owned configuration's `callers` list (`caller_not_permitted`).

```
helper-op/1
{ op:     "user.create" | "user.remove" | "user.list" | "fw.apply" | "fw.remove",
  op_id:  string,                       // idempotency key chosen by the caller
  args:   { … operation arguments … },
  grants: [ … ] }                       // v2 (§6); ignored before

helper-result/1
{ op_id, ok: true, result: { … } }
| { op_id, ok: false, error: { code, message } }
```

## 3. Users (v0)

### 3.1 Names and identifiers

- Agent users are named `cur-a-<label>`, service users `cur-s-<name>`. `label` and `name` MUST match `[a-z0-9][a-z0-9-]{0,22}` (`label_invalid`).
- UIDs and GIDs come from configured ranges, one for agents and one for services, chosen per OS so that the users stay hidden from login windows and do not collide with system accounts. The helper picks the lowest free value in the range after checking the directory service and its own ledger; it never accepts a UID from the caller.
- Each user gets its own primary group with the same name and number.

### 3.2 `user.create`

```
args:   { kind: "agent" | "service", label: "dev-7f3" }
result: { username: "cur-a-dev-7f3", uid: 612, gid: 612, home: "<agents root>/cur-a-dev-7f3",
          generation: "g-0193" }
```

The account has no password and no login shell (`/usr/bin/false` on macOS, `/usr/sbin/nologin` on Linux). The home is created from a fixed template under a fixed root (agents and services have separate roots), owned by the user, mode 0700. The helper refuses when the name or the home path already exists and is not in its ledger (`user_exists_unmanaged`, `home_exists_unmanaged`).

- **macOS:** records are created with `dscl` in the local node (`UniqueID`, `PrimaryGroupID`, `UserShell`, `NFSHomeDirectory`, `IsHidden`, a disabled password), not with `sysadminctl`, so that no FileVault or secure-token prompt can occur; such users can never unlock the disk or log in, which is intended.
- **Linux:** `useradd` and `groupadd` with explicit `--uid`/`--gid`, `--no-create-home`, `--shell /usr/sbin/nologin`; the home is created by the helper.

### 3.3 `user.remove`

```
args:   { username: "cur-a-dev-7f3", home: "archive" | "delete" }
result: { username, removed_at, archive?: path }
```

The helper removes only users recorded in its ledger as created by it (`user_not_managed`). It first stops every process of that UID and verifies that none remain (`processes_remain`), removes the user's firewall rules (§5), archives the home into a root-owned archive directory or deletes it, removes the user and group, and marks the ledger entry removed. A generation is never reused.

### 3.4 `user.list` and the ledger

The ledger is a root-owned, world-readable file with one record per created account: `{ username, uid, gid, kind, label, generation, created_at, created_by (caller UID), removed_at? }`. It contains no secrets. Other components (the broker) read it directly to check that a UID still belongs to the account generation they bound (`user.list` returns the same data through the helper).

## 4. Journal, rollback and idempotency

- Every operation writes a root-owned journal of planned and completed steps before acting.
- On failure, completed steps are undone in reverse order; if the helper dies mid-operation, the next invocation first replays the journal: it completes a removal or rolls back a creation, then answers the new request.
- A request whose `op_id` matches a completed operation returns that operation's result without acting again; a matching `op_id` with different arguments is refused (`op_id_conflict`).
- Paths are built only from fixed templates; every filesystem step uses `lstat` and refuses symbolic links (`path_is_link`); no path from the request is ever used.
- Every operation appends one line to a root-owned audit log: time, caller UID, operation, arguments, result.

## 5. Firewall per user (v1)

```
fw.apply  args: { username, allow: [ { proto: "tcp", host: "127.0.0.1", port: 18080 } ] }
fw.remove args: { username }
```

- The rules block all outbound traffic of the user's UID except the listed loopback destinations (its network profile's local proxy). Unix-domain sockets (the broker, the message bridge) are not affected by packet filters and stay reachable.
- **macOS:** one `pf` anchor per user under a helper-owned parent anchor, using `user <uid>` rules; the parent anchor is loaded once at installation.
- **Linux:** one nftables table owned by the helper with `meta skuid <uid>` rules.
- The allow list is derived by the caller from the agent's network profile; the helper validates that every host is a loopback address and every port is in a configured range (`fw_target_not_allowed`).
- Whether `pf` matches outgoing UDP and IPv6 by user on every supported macOS release is verified in the qualification suite before v1 ships.

This is what makes a network profile enforced instead of cooperative: an agent's process can reach the network only through its proxy, and an account that requires a profile (broker §5.5) can only be used through it.

## 6. Grant verification (v2)

In the platform design the helper is a handler process: it does not trust its caller and re-verifies the signed authority behind each request (`user.*` and `fw.*` actions, with a chain to a root). v2 adds that check with the same verification library the broker uses; the sudoers caller list then remains as defence in depth. Until v2, authority is the sudoers rule plus the `callers` list.

## 7. How it fits

| Caller | Operations | What it does next |
|---|---|---|
| dispatcher (v0: a Curator command) | `user.create`, `user.remove`, `fw.apply`, `fw.remove` | binds the new UID in the broker; registers it with the key keeper (later); starts the agent through the session host |
| platform installer | `user.create { kind: service }` | installs the broker (and other services) under the created users |
| broker | reads the ledger | checks UID generations before leasing |

## 8. Installation

The installer places the binary, the root-owned configuration (`callers`, UID and GID ranges, home roots, archive directory, allowed proxy ports), the ledger, the journal and audit directories, the sudoers rule and, for v1, the parent firewall anchor. Uninstall refuses while managed users exist unless forced by an operator.

## 9. Qualification

Tests that create users or rules run only on hosted CI runners (macOS and Ubuntu runners allow `sudo`), never on a developer's or a production host:

- create, list, remove, idempotent repeat, `op_id` conflict;
- refusals: invalid label, unmanaged user or home, symbolic link in any path component, caller not permitted, invocation without sudo;
- crash recovery: an operation killed after each step is completed or rolled back by the next invocation;
- v1: traffic from the user to a non-allowed destination is dropped, to the allowed proxy passes, other users are unaffected, rules are removed with the user.

## 10. Versions

| Version | Content |
|---|---|
| v0 | users: create, remove, list; ledger; journal and rollback; audit; macOS and Linux |
| v1 | per-user firewall rules (`pf` and nftables) |
| v2 | grant verification; parent read access to a child's home (ACL); sandbox profiles for donor machines |

## 11. Open questions

1. Home roots and whether a parent orchestrator gets read access to its children's homes (ACL) in v0 or v2.
2. Archive or delete homes by default on removal.
3. UID and GID ranges per OS.
4. Windows (later).

## Appendix A. Error codes

`not_invoked_by_sudo`, `caller_not_permitted`, `request_too_large`, `request_invalid`, `op_unknown`, `op_id_conflict`, `label_invalid`, `range_exhausted`, `user_exists_unmanaged`, `home_exists_unmanaged`, `user_not_managed`, `processes_remain`, `path_is_link`, `step_failed_rolled_back`, `fw_target_not_allowed`, `fw_unsupported`.
