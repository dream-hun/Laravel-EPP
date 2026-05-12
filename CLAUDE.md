# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Laravel package (`ywatchman/laravel-epp`) that provides an EPP (Extensible Provisioning Protocol) client for domain registration, primarily targeting SIDN (the Dutch domain registry for `.nl` domains). It is currently being rewritten to remove the `metaregistrar/php-epp-client` dependency.

## Commands

```bash
# Run all tests
./vendor/bin/pest

# Run a single test file
./vendor/bin/pest tests/SomeTest.php

# Lint / fix code style
./vendor/bin/pint

# Install dependencies
composer install
```

## Architecture

### Entry Point

`Epp` (`src/Epp.php`) is the main class. It manages the raw TCP/SSL socket connection to an EPP server. It handles session lifecycle: `start()` opens the socket, `login()` sends `Hello` then `Login` commands, and `logout()` / `__destruct()` closes the session. `sendRequest($xml)` and `read()` are the low-level I/O methods — EPP frames are length-prefixed with a 4-byte big-endian integer.

The `Facade` (`src/Facade.php`) and `ServiceProvider` (`src/ServiceProvider.php`) wire the package into Laravel under the `Epp` alias. Config is published from `config/epp.php`, which holds a `registrars` array keyed by registrar name (e.g. `sidn`).

### XML Command Layer (`src/Support/Xml/Commands/`)

All outbound EPP messages are built here using PHP's `DOMDocument`. The hierarchy is:

- `XmlHelper` — wraps `DOMDocument`, provides `createElement` and `addNode`
- `Command extends XmlHelper` — adds the `<command>` wrapper, `__toString()` serialises to the full `<epp>` XML string
- Concrete commands extend `Command`:
  - `Session/` — `HelloCommand`, `LoginCommand`, `LogoutCommand`
  - `Domain/` — `CheckCommand`, `CreateCommand`, `TransferCommand`, `UpdateCommand`
  - `Contact/` — `CheckCommand`, `CreateCommand`, `UpdateCommand`
  - `Host/` — `CheckCommand`, `CreateCommand`

`__toString()` on a command produces the complete XML string ready to send.

### Traits (`src/Support/Traits/Commands/`)

Mixed into commands to add optional capabilities:

- `HasDnssec` — adds DNSSEC key data to `secDNS:create` / `secDNS:update` extension nodes
- `HasExtensions` — generic extension support
- `HasScheduledDeletion` — SIDN-specific scheduled deletion extension
- `ProvidesCheckCommand` / `ProvidesContactCommand` — reusable check/contact node generation

### Extensions (`src/Support/Xml/Extensions/`)

Registry-specific EPP extensions. `Extension` is the base; `SidnExtension` sets the `sidn-ext-epp` prefix. `ContactExtension` handles SIDN-specific contact fields like `legalForm`.

### Models & Transformers

`Model` (`src/Models/Model.php`) is a lightweight value object with a `$columns` whitelist and magic `__get`/`__set` that store attributes in `$attributes`. `Domain`, `Contact`, and `Nameserver` extend it.

`Transformer` (`src/Transformers/Transformer.php`) is an abstract base; `ContactTransformer` maps a `Contact` model to the nested array structure expected by EPP `contact:create` XML. The `HasAuthentication` trait appends `authInfo` with a placeholder password.

### Response Parsing (`src/Responses/Response.php`)

`Response` wraps raw XML from the server using Symfony's `DomCrawler`. It extracts the result `code`, `msg`, `svTRID`, and optional `clTRID`. A response is considered successful when `code === '1000'`.

### Config

`config/epp.php` supports multiple registrars under `registrars.*`. Each entry requires `username`, `password`, `hostname`, and `port` (default 700). Set `EPP_DEBUG=true` to dump raw XML to stdout during socket I/O.

## Key Conventions

- New XML commands go under `src/Support/Xml/Commands/{Resource}/` and extend `Command`. Implement `__toString()` to build the `<epp>` document and `get*Node()` helpers for the inner structure.
- Registry-specific behaviour belongs in extensions under `src/Support/Xml/Extensions/Sidn/` or traits under `src/Support/Traits/Commands/`.
- Response parsing subclasses go under `src/Responses/` extending `Response`, using the inherited `$this->crawler` (a Symfony `DomCrawler` instance) to extract resource-specific data.
- The package namespace is `YWatchman\LaravelEPP`, PSR-4 autoloaded from `src/`.
