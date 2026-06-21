# OpenSSH Enhanced — Security Enhancements

This repository is a fork of [openssh-portable](https://github.com/openssh/openssh-portable) that adds
security hardening options not yet merged upstream.

## Features

### 1. `DisableTrivialAuth` (ssh_config — client)

**Purpose:** Prevents the SSH client from accepting a successful authentication that never
involved a real challenge-response exchange.

**Background:** The SSH protocol allows a server to accept the `none` auth method, or in
theory to send a success reply without ever issuing a real challenge. A misconfigured or
malicious server could therefore authenticate a client without actually verifying its
identity. This option detects such situations client-side.

**How it works:** The client tracks an `is_trivial_auth` flag that starts as `1` (trivial).
Each real authentication step resets it to `0`:

- GSSAPI: reset after a GSSAPI token has been successfully exchanged with the server
- Password: reset as soon as the client enters the password authentication method (before
  reading the server prompt)
- Keyboard-interactive: reset once for each prompt received from the server; stays trivial
  if the server sends zero prompts
- Public key: reset after a probe or signed request is successfully sent to the server

If authentication succeeds but `is_trivial_auth` is still `1`, the connection is aborted
with `"Trivial authentication disabled."`.

**Configuration (`~/.ssh/config` or `/etc/ssh/ssh_config`):**

```
DisableTrivialAuth yes
```

Default: `no`

---

### 2. `PubkeyDisablePKCheck` (sshd_config — server)

**Purpose:** Disables the public key pre-authentication probe to prevent public key
enumeration attacks.

**Background:** The standard SSH public key authentication protocol works in two steps:

1. The client sends a probe containing only the public key (no signature) asking the server
   whether it would accept this key.
2. If the server responds with `SSH2_MSG_USERAUTH_PK_OK`, the client signs and sends the
   full authentication request.

This two-step design leaks information: an attacker can probe arbitrary public keys against
the server to discover which keys are authorised, without needing to possess the corresponding
private key.

**How it works:** With `PubkeyDisablePKCheck yes`, the server skips the probe lookup entirely
(`goto done` before `user_key_allowed()`). The client receives no confirmation and must send
a signed request directly. The server then validates the signature and the key in a single
step.

**Configuration (`/etc/ssh/sshd_config`):**

```
PubkeyDisablePKCheck yes
```

Default: `no`

---

### 3. `PubkeyDisablePKCheck` (ssh_config — client)

**Purpose:** Lets the client skip the probe step and go straight to signing, matching a
server that has `PubkeyDisablePKCheck yes`.

**How it works:** The client skips the preliminary unsigned probe and sends the signed
`SSH_MSG_USERAUTH_REQUEST` directly. This only takes effect when `IdentitiesOnly yes` is
also set — without it, keys whose public component is already loaded (e.g. from `.pub` files)
still trigger a probe, because the client cannot guarantee the private key is available
without trying.

**Configuration (`~/.ssh/config`):**

```
IdentitiesOnly yes
PubkeyDisablePKCheck yes
```

Default: `no`

---

## Interaction between the two features

| Scenario | Result |
|---|---|
| Server has `PubkeyDisablePKCheck yes`, client does not | Client sends probe, server ignores it and returns no confirmation; client then signs and authenticates normally. |
| Client has `PubkeyDisablePKCheck yes` + `IdentitiesOnly yes`, server does not | Client skips probe and sends signature directly; server validates as normal. |
| Both sides have `PubkeyDisablePKCheck yes` (client also sets `IdentitiesOnly yes`) | No probe round-trip at all; one fewer network exchange. |
| Client has `DisableTrivialAuth yes` | Any auth that completes without a real challenge causes a fatal client-side error. |

## Security rationale

These features are intended for high-security or auditing environments where:

- Operators want to prevent any information leakage about which SSH keys are authorised
  on a server.
- Security tooling (e.g. SSH-MITM) must not be able to silently authenticate a client
  without a genuine credential exchange.
