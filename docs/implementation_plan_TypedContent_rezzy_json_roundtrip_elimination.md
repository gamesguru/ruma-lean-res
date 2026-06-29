# TypedContent: Eliminate JSON Value Lookups in Auth Hot Path

## Goal

Add `typed_content: Option<TypedContent>` to `LeanEvent`, replacing `content.get("field").as_str()` calls with direct field access. Callers with typed data (continuwuity) set it; JSON-only callers (CLI, tests) leave it `None` and use the existing `content: Value` fallback.

**Phase 1 is the permanent design** — no Phase 2 removal of `content: Value`.

## Proposed Changes

### Types ([src/types.rs](file:///run/media/shane/shane4tb-ent/repos/ruma-lean/src/types.rs))

#### [NEW] `TypedContent` struct + accessor helpers

```rust
#[derive(Debug, Clone, Default)]
pub struct TypedContent {
    // m.room.member
    pub membership: Option<String>,
    pub third_party_invite: Option<ThirdPartyInvite>,
    // m.room.power_levels
    pub users: Option<BTreeMap<String, i64>>,
    pub users_default: Option<i64>,
    pub events: Option<BTreeMap<String, i64>>,
    pub events_default: Option<i64>,
    pub state_default: Option<i64>,
    pub ban: Option<i64>,
    pub kick: Option<i64>,
    pub redact: Option<i64>,
    pub invite: Option<i64>,
    // m.room.join_rules
    pub join_rule: Option<String>,
    pub allow: Option<Vec<AllowCondition>>,
    // m.room.create
    pub creator: Option<String>,
    pub room_version: Option<String>,
    // m.room.third_party_invite
    pub public_key: Option<String>,
    pub public_keys: Option<Vec<PublicKey>>,
}
```

#### [MODIFY] [LeanEvent](file:///run/media/shane/shane4tb-ent/repos/ruma-lean/src/types.rs#L170-L195)

Add field:

```rust
pub typed_content: Option<TypedContent>,
```

Add typed accessor methods to `impl<Id> LeanEvent<Id>`:

```rust
pub fn get_membership(&self) -> Option<&str> { ... }
pub fn get_join_rule(&self) -> Option<&str> { ... }
pub fn get_users(&self) -> Option<&BTreeMap<String, i64>> { ... }
pub fn get_users_default(&self) -> Option<i64> { ... }
pub fn get_events(&self) -> Option<&BTreeMap<String, i64>> { ... }
pub fn get_events_default(&self) -> Option<i64> { ... }
pub fn get_state_default(&self) -> Option<i64> { ... }
pub fn get_ban(&self) -> Option<i64> { ... }
pub fn get_kick(&self) -> Option<i64> { ... }
pub fn get_invite(&self) -> Option<i64> { ... }
pub fn get_creator(&self) -> Option<&str> { ... }
```

Each accessor follows the pattern:

```rust
pub fn get_membership(&self) -> Option<&str> {
    if let Some(ref tc) = self.typed_content {
        return tc.membership.as_deref();
    }
    self.content.get("membership")?.as_str()
}
```

---

### Auth Module ([src/auth/mod.rs](file:///run/media/shane/shane4tb-ent/repos/ruma-lean/src/auth/mod.rs))

#### [MODIFY] All `content.get(...)` callsites → use typed accessors

| Line | Before                                                         | After                                                      |
| ---- | -------------------------------------------------------------- | ---------------------------------------------------------- |
| 295  | `pl_event.content.get("users").and_then(\|u\| u.as_object())`  | `pl_event.get_users()`                                     |
| 303  | `pl_event.content.get("users_default").and_then(as_i64)`       | `pl_event.get_users_default()`                             |
| 316  | `pl_event.content.get("events").and_then(\|e\| e.as_object())` | `pl_event.get_events()`                                    |
| 328  | `pl_event.content.get("state_default").and_then(as_i64)`       | `pl_event.get_state_default()`                             |
| 334  | `pl_event.content.get("events_default").and_then(as_i64)`      | `pl_event.get_events_default()`                            |
| 471  | `event.content.get("membership")`                              | `event.get_membership()`                                   |
| 477  | `ev.content.get("membership")`                                 | `ev.get_membership()`                                      |
| 509  | `ev.content.get("membership")`                                 | `ev.get_membership()`                                      |
| 522  | `ev.content.get("join_rule")`                                  | `ev.get_join_rule()`                                       |
| 555  | `pl_event.content.get("kick").and_then(as_i64)`                | `pl_event.get_kick()`                                      |
| 569  | `pl_event.content.get("invite").and_then(as_i64)`              | `pl_event.get_invite()`                                    |
| 582  | `pl_event.content.get("ban").and_then(as_i64)`                 | `pl_event.get_ban()`                                       |
| 667  | `content.get("membership")`                                    | stays (this is `auth_types_for_event`, takes raw `&Value`) |

---

### Other Modules

#### [MODIFY] [src/state_at.rs:109](file:///run/media/shane/shane4tb-ent/repos/ruma-lean/src/state_at.rs#L109)

`ev.content.get("membership")` → `ev.get_membership()`

#### [MODIFY] [src/sorting.rs:101,109](file:///run/media/shane/shane4tb-ent/repos/ruma-lean/src/sorting.rs#L101)

`pl_ev.content.get("users")` → `pl_ev.get_users()`
`pl_ev.content.get("users_default")` → `pl_ev.get_users_default()`

#### [MODIFY] [src/types.rs:406,430,447,468](file:///run/media/shane/shane4tb-ent/repos/ruma-lean/src/types.rs#L406)

`self.content.get("membership")` → `self.get_membership()`
`self.content.get("join_rule")` → `self.get_join_rule()`
`self.content.get("users")` → `self.get_users()`

#### [SKIP] `src/bin/rezzy/format.rs`, `src/bin/rezzy/utils.rs`

CLI display code — not hot path, keep `content.get()` as-is.

#### [SKIP] `auth_types_for_event` (line 667)

Takes `content: &Value` directly, not `&LeanEvent`. Leave as-is.

---

### Serialization

#### [MODIFY] `impl Serialize for LeanEvent`

Don't serialize `typed_content` — it's a runtime optimization, not a wire format.

#### [MODIFY] `impl Deserialize for LeanEvent<String>`

On deserialize from JSON, populate `typed_content` from `content` if the event type is a known auth-relevant type (`m.room.member`, `m.room.power_levels`, `m.room.join_rules`, `m.room.create`, `m.room.third_party_invite`).

---

### Tests

- All tests that construct `LeanEvent` manually: add `typed_content: None` field.
- Add targeted tests that construct events with `typed_content: Some(...)` and verify auth checks produce identical results.

## Open Questions

> [!IMPORTANT]
> The `get_users()` accessor returns `Option<&BTreeMap<String, i64>>` for typed content but the JSON fallback currently returns `Option<&serde_json::Map<String, Value>>`. The accessor needs to handle the type mismatch — either convert the JSON map on-the-fly (negating the performance win) or return a unified type. **Recommendation**: return `Option<&BTreeMap<String, i64>>` and only support it via `typed_content`. The JSON fallback for `users` would need to parse into a `BTreeMap` — but this only matters for callers that don't set `typed_content`.

## Verification Plan

### Automated Tests

```bash
cargo test --all-features
```

### Manual Verification

- Verify `test_busted_dag_resolution` and `test_unredacted_*` tests still pass with identical results.
