# Changelog

All notable changes to this collection are documented in this file.
The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `step_ca` role: install step-ca and step-cli from the official Smallstep APT repository,
  deploy an externally created intermediate CA, render `ca.json` from variables and run
  step-ca as a hardened systemd service.
