# Secure-IPA

Secure-IPA is a **modified version** of [FreeIPA](https://www.freeipa.org/), based on the upstream source at [Pagure](https://pagure.io/freeipa). It adds a `--ldaps-only` deployment mode and includes additional security-related patches. This fork is intended as a short-term option with the hope that the FreeIPA project will adopt these changes upstream.

---

## License and attribution

- **License:** This program is free software; you can redistribute it and/or modify it under the terms of the **GNU General Public License as published by the Free Software Foundation, version 3** (or at your option any later version). See the file [COPYING](COPYING) for the full license text.
- **Original work:** FreeIPA is Copyright (C) 2007 and later by the FreeIPA Project contributors and Red Hat, Inc. The upstream source is available at https://pagure.io/freeipa and https://github.com/freeipa/freeipa.
- **Modifications:** The modifications in this repository (including the `--ldaps-only` implementation and security patches) were made by **Scott Morrison**. This modified work is distributed in the hope that it will be useful, but **WITHOUT ANY WARRANTY**; without even the implied warranty of MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE. See the GNU General Public License for more details.
- **Not affiliated with the FreeIPA project:** This fork is an independent modification and is not officially endorsed or maintained by the FreeIPA project or Red Hat.

---

## Documentation

General FreeIPA documentation (concepts, architecture, standard deployment) applies to this fork. See the [FreeIPA documentation](https://www.freeipa.org/page/Documentation) and [upstream man pages](https://www.freeipa.org/page/Man_Pages). The sections below describe behavior that **differs** from upstream when using `--ldaps-only`.

---

## `--ldaps-only` mode

When `--ldaps-only` is used, the Directory Server (389-DS) listens **only on port 636 (LDAPS)**. Port 389 (plain LDAP) is disabled, and STARTTLS on port 389 is not used. All IPA components and clients use LDAPS to talk to the directory.

### Differences from upstream (main FreeIPA repository)

| Area | Upstream (default) | This fork with `--ldaps-only` |
|------|---------------------|-------------------------------|
| Directory Server | Listens on 389 (LDAP) and 636 (LDAPS) | Listens only on 636 (LDAPS); port 389 disabled |
| LDAP connections | `ldap://` with optional STARTTLS, or `ldaps://` | `ldaps://host:636` only |
| DNS SRV records | `_ldap._tcp` (389) published | `_ldap._tcp` omitted; `_ldaps._tcp` (636) published for discovery |
| Client discovery | Uses `_ldap._tcp`, then STARTTLS or LDAPS | Uses `_ldaps._tcp` and connects via LDAPS only |
| Replication | Can use port 389 + TLS or 636 | Uses port 636 with SSL transport only |
| CA/KRA (Dogtag) | DS connection to 389 or 636 | DS connection to 636 only |
| Config option | N/A | `ldaps_only = True` in `/etc/ipa/default.conf` |

### Server installation

- **First server:** Use `ipa-server-install ... --ldaps-only` (plus your other options). The Directory Server will be configured to listen only on 636; port 389 will be disabled after TLS is enabled.
- **Replicas:** Use `ipa-replica-install ... --ldaps-only`. The master must have been installed with `--ldaps-only`. The replica’s Directory Server will also listen only on 636.

**Ports:** With `--ldaps-only`, only **636** (LDAPS) is required for the directory. Upstream’s default also uses 389 (LDAP).

### Client installation

- Use `ipa-client-install ... --ldaps-only` when the IPA server was installed with `--ldaps-only`.
- The client is configured with `ldap_uri=ldaps://server:636` (or equivalent) and does not use plain LDAP or STARTTLS.
- If you use DNS discovery, the client looks for `_ldaps._tcp` SRV records; ensure the server is publishing them (this fork does so when `ldaps_only` is set).

### Configuration

- **`/etc/ipa/default.conf`:** When `--ldaps-only` is used, the installer adds `ldaps_only = True`. With that set, `ldap_uri` is derived as `ldaps://server:636` when not explicitly set.
- **Man pages:** See `ipa-server-install(1)`, `ipa-replica-install(1)`, and `ipa-client-install(1)` for the `--ldaps-only` option. See `default.conf(5)` for the `ldaps_only` and `ldap_uri` options.

### Tools

- **`ipa-replica-conncheck`:** Supports `--ldaps-only` so connectivity checks use only port 636.
- **`ipa-replica-manage`:** Uses the server’s `ldaps_only` configuration when performing replication-related operations.

---

## Security-related patches

In addition to `--ldaps-only`, this repository includes patches that are not (as of this writing) in the main FreeIPA repository:

- **Template evaluation:** The template subsystem’s use of `eval()` is restricted to a safe subset of characters (arithmetic only) to prevent code execution via crafted template content or variables.
- **Subprocess usage:** In `ipa_migrate`, external commands are run with argument lists and without the shell, to avoid command injection via hostname or similar input.
- **Randomness:** Cryptographically secure random is used where appropriate (e.g., AD trust confounder, sysrestore and upgrade backup naming).

These changes are best-effort security improvements and do not imply a full security audit of FreeIPA.

---

## Getting help

- For general FreeIPA usage and concepts: [FreeIPA documentation](https://www.freeipa.org/page/Documentation) and [upstream project](https://pagure.io/freeipa).
- For issues specific to this fork (e.g. `--ldaps-only` or the included patches), open an issue in this repository.
