# Proposal: Generic EventContent Architecture

## The Architectural Deadlock (The "Trilemma")

The root cause of architectural deadlocks in state resolution is attempting to solve a **polymorphic data problem** (Dynamic JSON CLI vs. Typed Library Structs) using a **monomorphic concrete struct** (`LeanEvent` containing both a JSON `Value` and an optional `TypedContent`).

This creates a Trilemma:

1. **Trap A (Allocation Overhead):** Eagerly deserializing JSON into a concrete `TypedContent` struct causes massive allocation overhead (`BTreeMap`, `String`) for every event, defeating the purpose of a "Lean" event.
2. **Trap B (JSON Lookup CPU):** Relying strictly on dynamic `serde_json::Value` lookups incurs heavy CPU penalties in the hot path.
3. **Trap C (Branching Overhead):** Allowing both and branching via `if let Some(ref tc) = self.typed_content` introduces severe branch prediction penalties in the tightest sorting and auth loops.

## Breaking the Trilemma: The Zero-Cost Abstraction

The Rust-native solution to this deadlock is **Static Dispatch via Generics (Compile-Time Monomorphization).** Instead of bridging the representations at runtime with branches, we extract the auth queries into a trait (`EventContent`) and make `LeanEvent` generic over its payload (`C: EventContent`).

This entirely shatters the Trilemma:

- **Trap A Defeated:** Consumers (like `conduwuit`) implement `EventContent` on a lightweight reference wrapper to their native database structures. Zero `BTreeMap` or `String` allocations are required.
- **Trap B Defeated:** Trait implementations execute direct, O(1) struct accesses. The JSON string-parsing penalty is quarantined exclusively to CLI/JSON tests.
- **Trap C Defeated:** Because the resolution engine is generic, the compiler **monomorphizes** the hot paths. The runtime branches disappear entirely at compile-time.

---

## Implementation Outline

### 1. Define the `EventContent` Trait

Establish the contract for what `rezzy` actually needs to resolve state, decoupling the mathematical engine from the memory layout. Flatten nested fields (like `ThirdPartyInvite`) into simple primitive queries to avoid creating intermediate struct definitions.

```rust
pub trait EventContent {
    fn get_membership(&self) -> Option<&str>;
    fn get_join_rule(&self) -> Option<&str>;
    fn get_user_power_level(&self, user: &str) -> Option<i64>;
    fn get_event_power_level(&self, event_type: &str) -> Option<i64>;
    fn get_users_default(&self) -> Option<i64>;
    fn get_events_default(&self) -> Option<i64>;
    fn get_state_default(&self) -> Option<i64>;
    fn get_ban(&self) -> Option<i64>;
    fn get_kick(&self) -> Option<i64>;
    fn get_invite(&self) -> Option<i64>;
    fn get_redact(&self) -> Option<i64>;
    fn get_creator(&self) -> Option<&str>;
    fn has_room_creator(&self, sender: &str) -> bool;
    fn has_additional_creator(&self, sender: &str) -> bool;

    // Flattens the `ThirdPartyInvite` struct hierarchy:
    fn get_third_party_invite_token(&self) -> Option<&str>;
}
```

### 2. Parameterize `LeanEvent`

Remove any bloated Option/Tuple pairings from the memory layout to restore cache density. Defaulting to `serde_json::Value` ensures existing CLI/Deserializer code does not break.

```rust
#[derive(Debug, Clone, Default)]
pub struct LeanEvent<Id = String, C = serde_json::Value> {
    pub event_id: Id,
    pub event_type: String,
    pub state_key: Option<String>,
    pub power_level: i64,
    pub origin_server_ts: u64,
    pub sender: String,
    pub prev_events: Vec<Id>,
    pub auth_events: Vec<Id>,
    pub depth: u64,

    // The payload data is completely generic
    pub content: C,
}
```

### 3. Delegate Accessors (Zero-Cost Passthrough)

By keeping `LeanEvent::get_membership()` but delegating it to the trait, we avoid modifying the core `auth` logic. The compiler will inline these proxy methods.

```rust
impl<Id, C: EventContent> LeanEvent<Id, C> {
    #[inline(always)]
    pub fn get_membership(&self) -> Option<&str> {
        self.content.get_membership()
    }
    // ... proxy the rest of the methods
}
```

### 4. Implement `EventContent` for `serde_json::Value`

Satisfy the CLI and test environments by isolating dynamic JSON lookups to a single trait implementation, strictly utilizing centralized string constants (e.g. `crate::event_types::FIELD_MEMBERSHIP`).

### 5. Propagate Generics Through the Hot Paths

Updating the engine signatures forces the compiler to generate the monomorphized execution paths.

```rust
// In src/auth/mod.rs
pub trait StateProvider<Id = String, C = serde_json::Value> {
    fn get_event(&self, event_type: &str, state_key: Option<&str>) -> Option<&LeanEvent<Id, C>>;
}

pub fn check_auth<Id: Clone, C: EventContent>(
    event: &LeanEvent<Id, C>,
    state: &impl StateProvider<Id, C>,
) -> Result<(), AuthError<Id>> { /* ... */ }

pub fn auth_types_for_event<C: EventContent>(
    event_type: &str,
    sender: &str,
    state_key: Option<&str>,
    content: &C,
    version: StateResVersion,
) -> Vec<(String, String)> {
    // Use content.get_third_party_invite_token() here
}

// In src/resolve.rs
pub fn resolve_lean<Id, C, S1, S2>(
    unconflicted_state: BTreeMap<(String, Option<String>), Id>,
    conflicted_events: HashMap<Id, LeanEvent<Id, C>, S1>,
    auth_context: &HashMap<Id, LeanEvent<Id, C>, S2>,
    version: StateResVersion,
) -> BTreeMap<(String, Option<String>), Id>
where
    Id: EventId,
    C: EventContent,
{ /* ... */ }
```

_(Note: The custom `Deserialize` implementation is restricted strictly to `LeanEvent<String, serde_json::Value>` so tests and the CLI keep parsing out of the box)._
