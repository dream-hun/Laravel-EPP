# Laravel-EPP Project Overview

## What It Is

A **Laravel package** that implements the **EPP (Extensible Provisioning Protocol)** — the standard protocol domain registrars use to manage domain names, contacts, and nameservers. It is specifically tailored for **SIDN**, the Dutch Internet registry managing `.nl` domains, with SIDN-specific extensions baked in.

- **License**: MIT
- **PHP**: ^8.2 | **Laravel**: ^11.40
- **Status**: Active rewrite (removing the old `metaregistrar/php-epp-client` dependency)
- **Packagist**: `ywatchman/laravel-epp`

---

## Architecture

```
src/
├── Epp.php                  ← Core: SSL socket connection + framing
├── Facade.php               ← Laravel facade alias: Epp::
├── ServiceProvider.php      ← Publishes config
├── Contracts/               ← Interfaces: Transformable, IsContact
├── Exceptions/              ← EppException (typed error codes)
├── Models/                  ← Plain value objects (no Eloquent)
│   ├── Domain               ← sld, tld, name
│   ├── Contact              ← All contact fields + legalForm (SIDN)
│   └── Nameserver           ← name, address (IPv4/IPv6)
├── Transformers/            ← Converts models → EPP-structured arrays
│   └── ContactTransformer
├── Responses/               ← Parse raw XML responses (Symfony DomCrawler)
│   ├── Response             ← Base: code, message, trID, succeeded
│   ├── Contact/{Check,Create}Response
│   ├── Domain/{Check,Create,Transfer,Update}Response
│   ├── Host/{Check,Create}Response
│   └── Registries/Sidn/Domain/{Create,Update}Response
└── Support/
    ├── Xml/
    │   ├── XmlHelper        ← DOMDocument wrapper
    │   ├── Commands/        ← Build outbound XML EPP commands
    │   │   ├── Session: Hello, Login, Logout
    │   │   ├── Contact: Check, Create, Update
    │   │   ├── Domain: Check, Create, Update, Transfer
    │   │   └── Host: Check, Create
    │   ├── Extensions/Sidn/ ← SIDN-specific XML extension nodes
    │   └── Objects/         ← Contact role objects (Admin, Tech, Billing)
    └── Traits/Commands/
        ├── HasDnssec        ← secDNS-1.1 extension support
        ├── HasScheduledDeletion  ← SIDN scheduled delete extension
        ├── HasExtensions
        ├── ProvidesCheckCommand
        └── ProvidesContactCommand
```

---

## How the Connection Works

`Epp.php` opens a **raw SSL socket** to the EPP server (default: port 700) and communicates using **EPP's length-prefixed framing** — a 4-byte big-endian header encoding the total packet length, followed by the XML body. The flow is:

1. `start()` → opens socket, reads server greeting
2. `login()` → sends `<hello>` then `<login>` with credentials
3. Send commands → `sendRequest($xml)` writes the framed packet, reads the response
4. `logout()` / destructor → sends `<logout>`, closes socket

---

## Supported EPP Operations

| Object | Operations |
|--------|-----------|
| Session | Hello, Login, Logout |
| Contact | Check, Create, Update |
| Domain | Check, Create, Update, Transfer |
| Host (Nameserver) | Check, Create |

---

## SIDN-Specific Extensions

The package hardcodes SIDN registry extensions:

- **`sidn-ext-epp`** — contact `legalForm` / `legalFormNo` fields on contact create
- **`secDNS-1.1`** — DNSSEC key data (flags, protocol, algorithm, pubKey) on domain create/update
- **`sidn-ext-epp-scheduled-delete-1.0`** — scheduled domain deletion with operations: `setDate`, `setDateToEndOfSubscriptionPeriod`, `cancel`

---

## Configuration (`config/epp.php`)

```php
'registrars' => [
    'sidn' => [
        'username' => env('SIDN_USERNAME'),
        'password' => env('SIDN_PASSWORD'),
        'hostname' => env('SIDN_HOSTNAME'),  // e.g. drs.domain-registry.nl
        'port'     => env('SIDN_PORT', 700),
        'timeout'  => env('SIDN_TIMEOUT', 30),
    ],
],
'debug' => env('EPP_DEBUG', false),  // dumps raw XML to stdout
```

Multiple registrars can be configured; the `Epp` constructor accepts a registrar key.

---

## Notable Issues / Tech Debt

1. **`read()` returns Exception objects instead of throwing** (`src/Epp.php:133-151`) — `return new Exception(...)` is silently ignored by callers instead of being thrown.
2. **No tests** — `pestphp/pest` and `orchestra/testbench` are in dev dependencies but no test directory exists.
3. **SIDN namespace hardcoded in base `Command::__toString()`** — `xmlns:sidn-ext-epp` is always emitted even for hypothetical non-SIDN registries.
4. **`guzzlehttp/guzzle`** is listed as a dependency but not visibly used anywhere in the current codebase.
5. **`clTRID` (client transaction ID)** is partially implemented — only `Domain\CreateCommand` uses it; the base `Command` class has a `// Todo: clTRID` comment.
6. **Debug output via `echo`** — debug mode prints raw XML to stdout rather than using Laravel's logger.
7. **`illuminate/database`** is required but no Eloquent models are used — the package uses its own plain `Model` value objects.

---

## Current Uncommitted Changes

The `develop` branch has modifications to `composer.json`, `Epp.php`, `Facade.php`, `ServiceProvider.php`, and `ContactTransformer.php` — consistent with the recent Laravel 11 upgrade commit in the history.
