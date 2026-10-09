# curator-host-helper

A minimal privileged helper for agent systems. It runs as root for one command at a time, through a sudoers rule that admits only its absolute path, and performs a closed set of host operations: creating and removing per-agent and per-service OS users, and (next) installing per-UID firewall rules so that an agent's traffic can leave only through its network profile.

**Status: design in progress.** Nothing is implemented yet. The specification will live in `spec/`.

## Principles

- A closed operation schema read from stdin (`helper-op/1`), never free-form arguments.
- The helper chooses user names and UIDs from its own ranges; callers pass only a validated label.
- It removes only accounts it created itself (a root-owned ledger), and never touches pre-existing accounts.
- Every operation is journaled and rolled back on failure, and leaves an audit line naming the caller.
- Later: it re-verifies the signed grant behind each request instead of trusting the caller.

## How it fits

- The dispatcher (or, until it exists, a Curator command) calls the helper to create an agent's OS user, then binds that user in curator-credential-broker and registers it with the key keeper.
- curator-credential-broker runs under a service user the helper creates.
