# 0017. Every port has a PHP twin, and binding it to PHP is the override

- **Status:** Accepted, 2026-11-18
- **Decided by:** the project owner; the rule for stateful ports by the designer

## Context

PHP may still implement a contract and override the Rust one. With no override, a call must stay in Rust.

## Decision

- A port is a Rust trait with a **PHP twin**: a PHP interface with the same methods, generated from the trait so the
  two cannot drift.
- A **PHP adapter** of a port is a Rust struct that implements the trait by calling a PHP object that implements the
  twin. Values are converted at that boundary only.
- **Binding a port to a PHP adapter is the override flag.** The container records, per key, whether the current
  binding is PHP. Resolving a Rust binding never enters PHP. Hot ports cache their resolved adapter, and the cache is
  dropped when the binding changes.
- **A stateful port** (a store, such as `HookStore`) is bound to a PHP adapter in the app's configuration
  (`php.bindings`), installed at request start before any PHP runs. Rebinding it from PHP after it has been used in
  the request is refused, naming the port. Stateless ports may be rebound from PHP at any point.

## Consequences

- A PHP store sees every registration from the start of a request.
- The twins come from `#[php_port]`.

## Patterns

- **Adapter:** the PHP adapters.
- **Strategy:** a port's adapter, chosen by binding.
