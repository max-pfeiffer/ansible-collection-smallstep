# max_pfeiffer.smallstep.step_ca

Install and manage a [Smallstep step-ca](https://smallstep.com/docs/step-ca/) certificate
authority on Debian based hosts, including Raspberry Pi OS.

The role:

1. Installs pinned versions of `step-ca` and `step-cli` from the official Smallstep APT repository
   (amd64, arm64, armhf and more).
2. Creates a dedicated `step` system user.
3. Deploys an **externally created** root CA certificate and intermediate CA certificate and key.
   The role never creates a root CA and never needs the root CA key.
4. Renders `ca.json` entirely from variables. Changes made on the host are overwritten on the next run.
5. Runs `step-ca` as a systemd service, hardened as in Smallstep's
   [production guide](https://smallstep.com/docs/step-ca/certificate-authority-server-production).
6. Waits until the `/health` endpoint of the CA responds.

Remote provisioner management (`enableAdmin`) is not supported, because it lets `step-ca` change
its own configuration, which conflicts with managing `ca.json` declaratively.

## Requirements

- Debian 12 or 13, or a distribution derived from them (Raspberry Pi OS, Ubuntu)
- systemd
- `become: true`

## File layout

| Path | Owner | Mode | Content |
|---|---|---|---|
| `/etc/step-ca/certs/root_ca.crt` | root:step | 0644 | Root CA certificate |
| `/etc/step-ca/certs/intermediate_ca.crt` | root:step | 0644 | Intermediate CA certificate |
| `/etc/step-ca/secrets/intermediate_ca_key` | root:step | 0640 | Encrypted intermediate CA key |
| `/etc/step-ca/secrets/password.txt` | root:step | 0640 | Intermediate CA key password |
| `/etc/step-ca/config/ca.json` | root:step | 0640 | step-ca configuration |
| `/etc/step-ca/config/defaults.json` | root:step | 0644 | step-cli defaults, so `STEPPATH=/etc/step-ca step ca health` works |
| `/etc/step-ca/db` | step:step | 0700 | Database (`bbolt.db` with `bbolt`), the only path writable by `step-ca` |

## Creating the PKI offline

Create the root CA on a trusted machine that is not the CA server, and keep the root key offline.
Restrict the intermediate CA with name constraints, so it can only issue certificates for your
domains, even if the CA server is compromised:

```shell
step certificate create "Home Lab Root CA" root_ca.crt root_ca_key --profile root-ca

cat > intermediate.tpl <<'EOF'
{
  "subject": {{ toJson .Subject }},
  "keyUsage": ["certSign", "crlSign"],
  "basicConstraints": {"isCA": true, "maxPathLen": 0},
  "nameConstraints": {
    "critical": true,
    "permittedDNSDomains": ["k8s.internal", "ca.lan"]
  }
}
EOF

step certificate create "Home Lab Intermediate CA" intermediate_ca.crt intermediate_ca_key \
  --template intermediate.tpl --ca root_ca.crt --ca-key root_ca_key
```

Include the DNS name of the CA server itself in `permittedDNSDomains`, because `step-ca` issues
its own TLS certificate with the intermediate CA.

## Role variables

All variables are validated by [`meta/argument_specs.yml`](meta/argument_specs.yml), which also
documents them in full. The most important ones:

| Variable | Default | Description |
|---|---|---|
| `step_ca_root_cert` | required | PEM root CA certificate |
| `step_ca_intermediate_cert` | required | PEM intermediate CA certificate |
| `step_ca_intermediate_key` | required | Encrypted PEM intermediate CA key, keep it in Ansible Vault |
| `step_ca_intermediate_key_password` | required | Intermediate CA key password, keep it in Ansible Vault |
| `step_ca_dns_names` | required | DNS names / IPs of the CA, the first one is used for `step_ca_url` |
| `step_ca_address` | `:9000` | Listen address |
| `step_ca_db_type` | `badgerv2` | `badgerv2` or `bbolt`, which needs less memory, for example on a Raspberry Pi |
| `step_ca_provisioners` | `[]` | Provisioners, written verbatim to `authority.provisioners` |
| `step_ca_claims` | `{}` | Authority wide claims, for example certificate durations |
| `step_ca_policy` | `{}` | Authority wide issuance policy, see [Issuance policy](#issuance-policy) |
| `step_ca_tls` | `{}` | TLS settings of the CA server |
| `step_ca_config_extra` | `{}` | Anything else, recursively merged into `ca.json` |
| `step_ca_resolver` | `""` | DNS resolver for ACME validation (`--resolver`) |
| `step_ca_wait_for_time_sync` | `true` | Start after `time-sync.target` |
| `step_ca_version` | pinned | `step-ca` package version |
| `step_ca_step_cli_version` | pinned | `step-cli` package version |

## Example: ACME DNS-01 CA for Kubernetes cert-manager

`step-ca` validates DNS-01 challenges by querying the resolver given in `step_ca_resolver`. Point it
at the DNS server that cert-manager updates via RFC 2136, here a Technitium DNS server on the same host.

```yaml
- name: Certificate authority
  hosts: ca
  become: true
  roles:
    - role: max_pfeiffer.smallstep.step_ca
      vars:
        step_ca_root_cert: "{{ lookup('ansible.builtin.file', 'pki/root_ca.crt') }}"
        step_ca_intermediate_cert: "{{ lookup('ansible.builtin.file', 'pki/intermediate_ca.crt') }}"
        step_ca_intermediate_key: "{{ vault_step_ca_intermediate_key }}"
        step_ca_intermediate_key_password: "{{ vault_step_ca_intermediate_key_password }}"
        step_ca_dns_names:
          - ca.lan
        step_ca_resolver: 127.0.0.1:53
        step_ca_claims:
          defaultTLSCertDuration: 720h
          maxTLSCertDuration: 2160h
        step_ca_policy:
          x509:
            allow:
              dns:
                - "*.k8s.internal"
            allowWildcardNames: true
        step_ca_provisioners:
          - type: ACME
            name: acme
            challenges:
              - dns-01
```

The ACME directory is then available at `https://ca.lan:9000/acme/acme/directory`.

## Issuance policy

`step_ca_policy` restricts the names `step-ca` issues certificates for. Two properties of step-ca
policies are easy to miss:

- **Authority level only.** A self-hosted `step-ca` reads policies only on the authority level of
  `ca.json` and silently ignores a `policy` on a provisioner, so the role refuses one.
- **A wildcard matches exactly one label.** `*.k8s.internal` allows `app.k8s.internal`, but neither
  `app.team.k8s.internal` nor `k8s.internal` itself. List every level you issue certificates for,
  for example `*.team.k8s.internal`.

`allowWildcardNames` defaults to `false`, so it must be enabled explicitly to issue wildcard
certificates like `*.k8s.internal`.

To allow all names below a domain at any depth, rely on the name constraints of the intermediate CA
instead (see [Creating the PKI offline](#creating-the-pki-offline)). Clients enforce them, and
`step_ca_policy` can stay empty.

## Time synchronization

Hosts without a hardware clock, like the Raspberry Pi, boot with a wrong time. With
`step_ca_wait_for_time_sync: true` the service is ordered after `time-sync.target`. This only
delays the start if a wait service is enabled, for example:

```shell
systemctl enable systemd-time-wait-sync.service   # systemd-timesyncd
systemctl enable chrony-wait.service              # chrony
```

## Health check

After starting the service, the role requests `{{ step_ca_url }}/health` from the CA host and
validates the TLS certificate against the root CA. The first name in `step_ca_dns_names` must
therefore resolve on the CA host itself. Disable the check with `step_ca_health_check: false`.

## License

MIT
