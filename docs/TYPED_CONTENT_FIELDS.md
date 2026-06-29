# Proposal: Typed Content Fields (`LeanEventTyped`)

## Problem

`LeanEvent` stores the event's content as `serde_json::Value`. Every auth check
in the hot path calls `ev.content.get("membership")`, `ev.content.get("users")`,
etc. — each of which requires hash lookups into a JSON object, string comparisons,
and type coercion at runtime.

Worse, downstream callers (e.g. continuwuity) already have typed access to these
fields via ruma deserialization. The current adapter pattern serializes typed
fields _back_ to `serde_json::Value`, and rezzy re-parses them — a completely
redundant JSON round-trip.

## Proposed Solution

### New Struct: `TypedContent`

Replace `content: serde_json::Value` with a pre-extracted struct:

```rust
/// Pre-extracted content fields used during auth checks and resolution.
///
/// Only fields that rezzy actually reads during state resolution are included.
/// All fields are `Option` — absent means "not present in the event content."
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

### Migration Strategy

#### Phase 1: Dual Support (Non-Breaking)

Add `TypedContent` as an **optional** field alongside `content`:

```rust
pub struct LeanEvent<Id = String> {
    // ... existing fields ...
    pub content: Value,
    /// Pre-extracted typed content. If present, auth checks use this
    /// instead of parsing `content`. Set by callers that already have
    /// typed access (e.g. ruma adapters).
    pub typed_content: Option<TypedContent>,
}
```

Auth checks become:

```rust
fn get_membership(ev: &LeanEvent) -> Option<&str> {
    // Fast path: typed content
    if let Some(ref tc) = ev.typed_content {
        return tc.membership.as_deref();
    }
    // Fallback: JSON
    ev.content.get("membership")?.as_str()
}
```

#### Phase 2: Primary (Breaking)

Once all downstream callers migrate, make `TypedContent` the primary storage
and remove `content: Value`:

```rust
pub struct LeanEvent<Id = String> {
    // ... existing fields ...
    pub content: TypedContent,
}
```

### Affected Files

| File                | Change                                                      |
| ------------------- | ----------------------------------------------------------- |
| `src/types.rs`      | Add `TypedContent` struct, update `LeanEvent`               |
| `src/auth/mod.rs`   | Replace all `ev.content.get(...)` with typed accessors      |
| `src/auth/v1.rs`    | Same                                                        |
| `src/resolve.rs`    | Minor — only reads content indirectly via auth              |
| `src/lattice.rs`    | `route_power_events` checks content for membership          |
| `tests/*.rs`        | Update test event construction                              |
| Downstream adapters | Change from `serde_json::to_value(...)` to field assignment |

### Content Fields Accessed by Rezzy

Exhaustive list of `ev.content.get(...)` calls across the codebase:

| Field                        | Event Type                  | Used In                            |
| ---------------------------- | --------------------------- | ---------------------------------- |
| `membership`                 | `m.room.member`             | Auth checks, power event routing   |
| `users`                      | `m.room.power_levels`       | Power level lookups                |
| `users_default`              | `m.room.power_levels`       | Default PL for unknown users       |
| `events`                     | `m.room.power_levels`       | Per-type PL requirements           |
| `events_default`             | `m.room.power_levels`       | Default PL for unknown event types |
| `state_default`              | `m.room.power_levels`       | Default PL for state events        |
| `ban`                        | `m.room.power_levels`       | Ban level threshold                |
| `kick`                       | `m.room.power_levels`       | Kick level threshold               |
| `redact`                     | `m.room.power_levels`       | Redaction level threshold          |
| `invite`                     | `m.room.power_levels`       | Invite level threshold             |
| `join_rule`                  | `m.room.join_rules`         | Join rule checks                   |
| `allow`                      | `m.room.join_rules`         | Restricted room checks             |
| `creator`                    | `m.room.create`             | Room creator for v1-v10            |
| `room_version`               | `m.room.create`             | Room version detection             |
| `third_party_invite`         | `m.room.member`             | 3PID invite auth                   |
| `public_key` / `public_keys` | `m.room.third_party_invite` | 3PID invite validation             |

### Performance Impact

- **Auth checks**: Eliminates ~10-15 `HashMap::get` + `Value::as_str()` calls
  per event during iterative auth checking. For a resolution with 1000 conflicted
  events, this is ~10K-15K hash lookups eliminated.
- **Adapter overhead**: Eliminates the `serde_json::to_value(ruma_content)` →
  `value.get("field").as_str()` round-trip. For continuwuity, this is the
  dominant cost in the PDU→LeanEvent conversion.
- **Memory**: `TypedContent` is likely smaller than `serde_json::Value` for most
  events since it doesn't store field names as heap-allocated strings.

### Risks

- **Breaking change** (Phase 2): All downstream callers must update event
  construction. Mitigated by the Phase 1 dual-support period.
- **Completeness**: If a future MSC adds new content fields that auth checks
  need, `TypedContent` must be extended. This is a maintenance burden vs the
  flexibility of `Value`.
- **Deserialization**: The custom `LeanEvent` deserializer must be updated to
  populate `TypedContent` from raw JSON. This is non-trivial but bounded.

### Derived Fields: `is_power_event`

Once `TypedContent` exists, power event classification becomes a trivial derived
method instead of requiring JSON introspection:

```rust
impl<Id> LeanEvent<Id> {
    /// Whether this event is a power event for the given resolution version.
    ///
    /// Power events affect the room's administrative state and are resolved
    /// first during state resolution. The definition varies by version:
    /// - V1/V2: all `m.room.member` events are power events
    /// - V2.1+: only ban/kick `m.room.member` events are power events
    pub fn is_power_event(&self, version: StateResVersion) -> bool {
        matches!(self.event_type.as_str(),
            "m.room.create" | "m.room.power_levels" | "m.room.join_rules"
        ) || match version {
            StateResVersion::V2_1 | StateResVersion::V2_1_1 | StateResVersion::V2_2 => {
                self.content.membership.as_deref() == Some("ban")
                    || self.content.membership.as_deref() == Some("leave")
                        && self.state_key.as_deref() != Some(&self.sender)
            }
            _ => self.event_type == "m.room.member",
        }
    }
}
```

This eliminates the need for DB-level `is_power_event` boolean columns, which
would be fragile because the definition is room-version-dependent. Computing it
from `TypedContent` at runtime is nanoseconds and always correct.
