# swarma-user-manager

Two minimal privileged components for agent systems. Each runs as root for one operation at a time, through a sudoers rule that admits only its absolute path with no arguments, and performs a closed set of host operations:

- **swarma-user-manager** creates, lists and retires per-agent and per-service OS accounts, and (next) installs per-account firewall rules so that an agent's traffic can leave only through its network profile;
- **swarma-user-launcher** starts one approved executor under one active agent account, so that an unprivileged dispatcher can run an agent as that account without general `sudo` rights.

**Status: design in progress.** Nothing is implemented yet. The specification is [`spec/helper.md`](spec/helper.md); diagrams are in [`diagrams/`](diagrams/).

## Principles

- Closed operation schemas read from stdin (`helper-op/1`, `launch-op/1`), never free-form arguments, commands or environment.
- Per-caller policy: installers manage service accounts, dispatchers manage only the agents they created.
- The helper chooses account names and UIDs from its own ranges; every creation gets a generation that is never reused.
- It acts only on accounts it created (a root-owned ledger), never on pre-existing accounts, and works on files through descriptors without following links.
- One lock serialises everything; creation rolls back on failure, retirement moves forward from its commit point; every operation leaves an audit line naming the caller.
- Later: it re-verifies the signed grant behind each request instead of trusting the configured policy alone.

## How it fits

- [swarma-dispatcher](https://github.com/relux-works/swarma-dispatcher), under its own service account, creates an agent's account with the helper, binds that account's generation in swarma-credential-broker, and starts the agent's executor through the launcher; the executor then obtains its credential lease from the broker itself.
- swarma-credential-broker runs under a service account the helper creates, and reads the helper's ledger to check generations.

## License

Apache License 2.0 — see [LICENSE](LICENSE) and [NOTICE](NOTICE). Authors: Ivan Oparin and Alexey Grigorev.
