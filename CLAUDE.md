# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Runtime and test runner is Bun (no build step, no linter configured).

- `bun run start` — run `src/main.ts` (wires the dependencies and places one sample order)
- `bun test` — run all tests
- `bun test tests/order.test.ts` — run a single test file; add `-t "<name pattern>"` to filter by test name
- `bun run typecheck` — `tsc --noEmit` (strict, plus `noUncheckedIndexedAccess` and `exactOptionalPropertyTypes`)

Imports use the `@/*` alias for `src/*` (see `tsconfig.json`).

## Architecture

A small Clean Architecture / ports-and-adapters teaching example around a single use case, "place an order". Dependencies point inward only: `adapters` → `application` → `domain`.

- `src/domain/` — entities and value objects (`Order`, `Money`, `EmailAddress`). Invariants are enforced here and violations throw `DomainRuleViolation`. No I/O and no imports from outer layers.
- `src/application/` — use cases (`PlaceOrder`) and the **ports** they depend on (`ports/`: `OrderRepository`, `PaymentGateway`, `IdGenerator`). The use case receives its ports via constructor injection. It takes primitive input (`PlaceOrderInput`), builds domain objects, charges the payment, then marks the order paid and saves it. A declined payment is returned as `{ ok: false, reason }`. Domain violations are thrown.
- `src/adapters/` — implementations of the ports (`persistence/`, `payment/`, `identity/`) plus the driving adapter `http/place-order-controller.ts`. The controller validates the untyped request body, then maps results to HTTP: 400 for a malformed body, 422 for a declined payment or a `DomainRuleViolation`, 201 on success. Unknown errors are rethrown.
- `src/main.ts` — composition root: the only place where concrete adapters are chosen and wired.
- `src/anti-example/coupled-checkout.ts` — deliberately coupled counterpart (HTTP, SQL, pricing and payment SDK in one function) kept for contrast. It is not part of the real design, so don't refactor it or import from it.

Tests in `tests/` use `bun:test`. They exercise the domain objects directly, the use case with the in-memory/configurable adapters, and the controller. No mocking framework is used. Tests swap in the adapters, e.g. `ConfigurablePaymentGateway` takes a boolean to approve or decline.
