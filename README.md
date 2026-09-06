[![Test aliasd](https://github.com/algo7/aliasd/actions/workflows/run-test.yml/badge.svg)](https://github.com/algo7/aliasd/actions/workflows/run-test.yml)

# Table of Contents

- [Table of Contents](#table-of-contents)
- [Description](#description)
- [Supported Operating Systems and Architectures](#supported-operating-systems-and-architectures)
- [User Guide](#user-guide)
  - [Configuration File](#configuration-file)
  - [Entry Validation](#entry-validation)
  - [Environment Variables](#environment-variables)
  - [Running the Server](#running-the-server)
  - [Running with Docker](#running-with-docker)
  - [HTTP API](#http-api)
  - [pfSense Setup](#pfsense-setup)
- [Maintainer Guide](#maintainer-guide)
  - [Project Structure](#project-structure)
  - [Building the Tool](#building-the-tool)
  - [Testing the Tool](#testing-the-tool)

# Description

**aliasd** serves named lists of IP addresses and CIDR prefixes over HTTP, in the plain-text, one-entry-per-line format a pfSense URL Table alias consumes.

The lists live in a single YAML file, one top-level key per alias. Every entry is validated when the server starts, so a malformed prefix aborts startup with a message naming the alias and the offending value.

# Supported Operating Systems and Architectures

| OS      | Architecture |
| ------- | ------------ |
| Linux   | x86_64       |
| macOS   | x86_64       |
| macOS   | arm64        |
| Windows | x86_64       |

# User Guide

## Configuration File

Each top-level key is an alias name and becomes the URL path it is served at. The example below is served at `/office-vpn` and `/blocklist`.

```yaml
office-vpn:
  - "10.8.0.0/24"
  - "10.9.0.0/24"

blocklist:
  - "203.0.113.0/24"
  - "198.51.100.17"
  - "2001:db8::/32"

# Both forms below define an alias with no entries and will return 404 when requested.
staging-allow: []
retired-hosts:
```

Entries are served in the order they are written.

Note that the file is read once at startup, so editing it requires a restart.

An alias with no entries is ignored and requesting it returns 404.

## Entry Validation

An entry is either a bare address or a CIDR prefix, IPv4 or IPv6. Two rules are enforced before the server will start:

- A prefix must be the network address. `10.8.0.5/24` is rejected, because it is ambiguous about whether you meant the subnet or the single host.
- IPv6 zone suffixes (`fe80::1%eth0`) are rejected — they are meaningless to a remote consumer.

A rejected prefix names the fix:

```
alias "office-vpn": "10.8.0.5/24" has host bits set: use 10.8.0.0/24 for the network or 10.8.0.5/32 for the single host
```

## Environment Variables

| Variable      | Default        | Description                                                                                                                                      |
| ------------- | -------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `ALIAS_FILE`  | `aliases.yaml` | Path to the YAML file holding the alias definitions.                                                                                             |
| `ALIAS_USER`  | `pfsense`      | HTTP Basic username.                                                                                                                             |
| `ALIAS_PASS`  | (unset)        | HTTP Basic password. **Unset disables authentication entirely** use for local development only. The server logs a warning at startup when it is. |
| `LISTEN_ADDR` | `:8080`        | Address the server listens on.                                                                                                                   |

## Running the Server

```bash
go build -o aliasd .
ALIAS_PASS=s3cret ALIAS_FILE=./aliases.yaml ./aliasd
```

## Running with Docker

The image does not bundle a config file, so mount one. `ALIAS_FILE` defaults to `aliases.yaml` relative to the image's `/` working directory.

```bash
make docker-build

docker run --rm -p 8080:8080 \
  -v "$PWD/aliases.yaml:/aliases.yaml:ro" \
  -e ALIAS_PASS=s3cret \
  aliasd:latest
```

## HTTP API

| Request                                             | Response                                  |
| --------------------------------------------------- | ----------------------------------------- |
| `GET /{name}` with valid credentials, alias exists  | `200`, `text/plain`, one entry per line   |
| `GET /{name}` with valid credentials, no such alias | `404`                                     |
| Any request with missing or wrong credentials       | `401` with a `WWW-Authenticate` challenge |

```bash
curl -u pfsense:s3cret http://localhost:8080/office-vpn
```

## pfSense Setup

Create a **URL Table (IPs)** alias under **Firewall** -> **Aliases** -> **Add** -> **Type URL(IPs)** pointing at the alias you want.
Credentials are supplied the usual way for a fetched URL:

```
https://user:password@aliasd.example.com/blocklist
```

Serve it over TLS or put it behind a reverse proxy; plain HTTP will shows the credentials in cleartext on the network.

# Maintainer Guide

## Project Structure

```bash
.
├── aliases.yaml    // Local alias definitions used for development; not baked into the image
├── Dockerfile      // Two-stage build onto distroless, running as nonroot (65532)
├── main.go         // Reads the environment, loads the config, wires the mux, starts the server
├── Makefile        // Build, test and Docker targets
└── internal
    ├── auth        // HTTP Basic middleware, constant-time credential comparison
    └── config      // YAML loading, entry validation, and the alias HTTP handler
```

`Config` is both the parsed config and the handler that serves it — it implements `http.Handler`, which is what keeps the alias-serving logic testable rather than buried in a closure in `main()`.

## Building the Tool

```bash
go build -o aliasd .   # current platform
make build-all         # all four supported targets, into bin/
make docker-build      # container image, tagged aliasd:latest
```

## Testing the Tool

```bash
make test              # go fmt, go vet, then tests with an HTML coverage report
go test ./internal/... # tests alone
go test -cover ./internal/...
```
