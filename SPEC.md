# SPEC: cartlib — a stateless shopping-cart library

A small, dependency-free Python library that demonstrates the **explicit state
handle** pattern: all application state lives in a store, and callers hold only
an opaque, signed token that references it. Because no state lives in the
process serving a call, *any* process holding the same store and secret can
serve *any* call — the pattern that lets services scale behind a plain
round-robin load balancer.

## Ground rules (apply to every work item)

- Python 3.11+, **standard library only** — no third-party runtime dependencies.
- Package layout: a `cartlib/` package at the repository root (with `__init__.py`).
- Tests: pytest-style test files under `tests/` (plain `assert`, no unittest
  classes required). Every work item that adds behavior ships tests for it.
- Docstrings on public functions; keep modules small and readable.
- Implementers: read this SPEC.md for your work item's details — your issue is a
  pointer to a section of this file.

## Work item 1 — Token codec (`cartlib/token.py`)

The signed handle. No dependencies on other work items.

- `encode_token(cart_id: str, secret: bytes) -> str` — returns an opaque,
  URL-safe string embedding the `cart_id` and an HMAC-SHA256 signature computed
  with `secret`.
- `decode_token(token: str, secret: bytes) -> str` — verifies the signature
  (constant-time comparison) and returns the `cart_id`.
- A tampered or malformed token raises `CartTokenError` (define it here).
- A client cannot forge a token it was never given, but tokens are not
  encrypted — the cart id is merely opaque, not secret. Say so in the docstring.
- Tests: round-trip works; tampering with any character raises `CartTokenError`;
  two different secrets don't accept each other's tokens.

## Work item 2 — Cart store (`cartlib/store.py`)

Where the state actually lives. No dependencies on other work items.

- A `CartStore` Protocol (typing.Protocol) with three methods:
  `create() -> str` (returns a new unique cart id),
  `get(cart_id: str) -> dict` (returns `{"id": ..., "items": [...]}`),
  `add_item(cart_id: str, name: str, qty: int) -> dict` (appends an item,
  returns the updated cart dict).
- `InMemoryCartStore` — a dict-backed implementation of that protocol.
- Unknown `cart_id` raises `CartNotFoundError` (define it here).
- Items are dicts: `{"name": str, "qty": int}`. Reject `qty < 1` with `ValueError`.
- Tests: create/get/add round-trip; unknown id raises; qty validation.

## Work item 3 — Cart operations (`cartlib/ops.py`)

The public API tying the codec and store together. **Depends on work items 1
and 2.**

- `create_cart(store, secret) -> dict` — creates a cart in the store and returns
  `{"cart_token": <encoded token>}`.
- `add_item(store, secret, cart_token, name, qty) -> dict` — decodes the token,
  adds the item, returns the updated cart dict **plus** the same `cart_token`.
- `get_cart(store, secret, cart_token) -> dict` — decodes and returns the cart
  dict plus the token.
- These functions hold no state of their own: given the same `store` and
  `secret`, two different "instances" (call sites, processes, threads) can serve
  interleaved calls for the same token. One test must demonstrate exactly that:
  create via one ops call, add via a "second instance" (a separate import or
  function reference), get via the first — same result.
- Invalid tokens surface `CartTokenError`; missing carts surface
  `CartNotFoundError` — no swallowing.

## Work item 4 — Demo script and README (`demo.py`, `README.md`)

The show-and-tell. **Depends on work item 3.**

- `demo.py` at the repo root, runnable with `python demo.py`:
  1. creates a cart, adds two items, reads it back — printing the token and the
     cart after each step;
  2. simulates a second instance serving a request for the same token (shared
     store, fresh function references) to show statelessness;
  3. tampers with one character of the token and shows the rejection
     (catch `CartTokenError`, print a friendly line).
- Rewrite `README.md` to describe cartlib: what the explicit-state-handle
  pattern is, the package layout, how to run the demo and the tests. Keep the
  note that this repo is an agent-operated sandbox.
