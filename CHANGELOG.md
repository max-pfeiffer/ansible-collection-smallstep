# Changelog

All notable changes to this collection are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Fixed

- `step_ca` role: `step_ca_db_type: bbolt` failed to start, because the data source was the
  database directory. bbolt now stores its database in `<step_ca_path>/db/bbolt.db`.
- `step_ca` role: the README example set a `policy` on the ACME provisioner, which a self-hosted
  step-ca silently ignores. The example now uses `step_ca_policy`, and the role refuses a policy on
  a provisioner.

### Changed

- `step_ca` role: `step_ca_db_type` accepts `badgerv2` and `bbolt` only, because the role always
  stores the database below `<step_ca_path>/db`.

### Added

- `step_ca` role: install step-ca and step-cli from the official Smallstep APT repository,
  deploy an externally created intermediate CA, render `ca.json` from variables and run
  step-ca as a hardened systemd service.
