# OCX Claude Code Harness Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a built-in Buzz harness that uses the Claude ACP adapter with a user-managed OpenCodex (OCX) proxy, enabling configured OpenRouter models in Claude Code.

**Architecture:** Extend the existing static preset metadata with a small dependency list and default environment map. The OCX preset runs `claude-agent-acp`; Buzz’s existing Claude adapter integration supplies the Claude executable, while the preset routes its API traffic to OCX’s loopback gateway. OCX owns provider credentials and lifecycle; Buzz neither reads OCX configuration nor starts its service.

**Tech Stack:** Rust, Tauri desktop managed-agent catalog, Agent Client Protocol, OpenCodex.

---

## File structure

- Modify: `desktop/src-tauri/src/managed_agents/discovery/presets.rs`
  - Define the OCX preset, retain its environment in the generated harness definition, and mark the preset unavailable unless both OCX and the Claude ACP adapter are on `PATH`.
  - Extend focused catalog tests in the same file.
- Modify: `desktop/src-tauri/src/managed_agents/types.rs`
  - Correct the catalog-field comment: preset definitions may carry immutable environment defaults.

### Task 1: Specify the OCX preset contract with failing catalog tests

**Files:**

- Modify: `desktop/src-tauri/src/managed_agents/discovery/presets.rs`

- [ ] **Step 1: Add an OCX contract test before the preset exists**

```rust
#[test]
fn ocx_preset_requires_ocx_and_claude_acp_and_routes_to_the_local_gateway() {
    let preset = PRESET_HARNESSES
        .iter()
        .find(|preset| preset.id == "ocx")
        .expect("OCX preset should be present");

    let available = preset_catalog_entry(preset, |command| match command {
        "claude-agent-acp" => Some(PathBuf::from("/usr/local/bin/claude-agent-acp")),
        "ocx" => Some(PathBuf::from("/usr/local/bin/ocx")),
        _ => None,
    });
    assert_eq!(available.availability, AcpAvailabilityStatus::Available);
    assert_eq!(available.command.as_deref(), Some("claude-agent-acp"));
    assert_eq!(
        available.definition_env.get("ANTHROPIC_BASE_URL").map(String::as_str),
        Some("http://127.0.0.1:10100")
    );
    assert_eq!(
        available
            .definition_env
            .get("CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY")
            .map(String::as_str),
        Some("1")
    );

    let missing_ocx = preset_catalog_entry(preset, |command| {
        (command == "claude-agent-acp").then(|| PathBuf::from("/usr/local/bin/claude-agent-acp"))
    });
    assert_eq!(missing_ocx.availability, AcpAvailabilityStatus::AdapterMissing);
    assert!(missing_ocx.command.is_none());
}
```

- [ ] **Step 2: Run the focused test to verify it fails**

Run:

```powershell
cargo test --manifest-path desktop/src-tauri/Cargo.toml ocx_preset_requires_ocx_and_claude_acp_and_routes_to_the_local_gateway --lib
```

Expected: FAIL because the `ocx` preset does not exist.

- [ ] **Step 3: Commit the failing test**

```powershell
git add desktop/src-tauri/src/managed_agents/discovery/presets.rs
git commit -m "test: define OCX harness contract"
```

### Task 2: Implement the smallest reusable preset metadata extension

**Files:**

- Modify: `desktop/src-tauri/src/managed_agents/discovery/presets.rs`
- Modify: `desktop/src-tauri/src/managed_agents/types.rs`

- [ ] **Step 1: Add immutable environment defaults and required executable metadata**

Add these fields to `PresetHarness`:

```rust
    env: &'static [(&'static str, &'static str)],
    required_commands: &'static [&'static str],
```

Set `env: &[]` and `required_commands: &[]` on every existing preset and on `ADAPTER_PRESET`.

- [ ] **Step 2: Make `preset_catalog_entry` reject a missing dependency**

After resolving `def.command`, treat any unresolved entry in `def.required_commands` as `AcpAvailabilityStatus::AdapterMissing`; set `command` and `binary_path` to `None`. Preserve the current command-missing behavior so existing presets remain unchanged.

- [ ] **Step 3: Pass immutable preset environment into the catalog and launch definition**

In both `preset_catalog_entry` and `preset_harness_definitions`, build the map from `preset.env`:

```rust
preset.env.iter().map(|(key, value)| ((*key).to_string(), (*value).to_string())).collect()
```

Assign it to `definition_env` in the catalog entry and to `env` in the generated `HarnessDefinition`. This reuses Buzz’s existing environment layering, where global/persona/agent values override preset defaults.

- [ ] **Step 4: Correct the stale catalog-field comment**

In `desktop/src-tauri/src/managed_agents/types.rs`, change the `definition_env` comment so it says immutable builtin and preset defaults may be present, while custom entries retain their saved definition environment.

- [ ] **Step 5: Add the OCX preset**

Append this definition to `PRESET_HARNESSES`:

```rust
    PresetHarness {
        id: "ocx",
        label: "OCX (Claude Code)",
        command: "claude-agent-acp",
        args: &[],
        env: &[
            ("ANTHROPIC_BASE_URL", "http://127.0.0.1:10100"),
            ("ANTHROPIC_AUTH_TOKEN", "opencodex-proxy"),
            ("CLAUDE_CODE_ENABLE_GATEWAY_MODEL_DISCOVERY", "1"),
        ],
        required_commands: &["ocx"],
        install_instructions_url: "https://opencodex.me/guides/claude-code/",
        install_hint: "Install OCX and the Claude ACP adapter, configure OpenRouter in OCX, then start `ocx service` (or run `ocx start`). Buzz does not manage OCX credentials or its service.",
        underlying_cli: Some("ocx"),
    },
```

The placeholder auth token intentionally selects OCX proxy mode when there is no Claude login. Users can override all three variables per agent if their OCX port or admission-token configuration differs.

- [ ] **Step 6: Run the focused test to verify it passes**

Run:

```powershell
cargo test --manifest-path desktop/src-tauri/Cargo.toml ocx_preset_requires_ocx_and_claude_acp_and_routes_to_the_local_gateway --lib
```

Expected: PASS.

- [ ] **Step 7: Commit the implementation**

```powershell
git add desktop/src-tauri/src/managed_agents/discovery/presets.rs desktop/src-tauri/src/managed_agents/types.rs
git commit -m "feat: add OCX Claude Code harness"
```

### Task 3: Verify catalog behavior and formatting

**Files:**

- Modify: `desktop/src-tauri/src/managed_agents/discovery/presets.rs`

- [ ] **Step 1: Add the complementary missing-adapter assertion**

Extend the Task 1 test with this case:

```rust
    let missing_adapter = preset_catalog_entry(preset, |command| {
        (command == "ocx").then(|| PathBuf::from("/usr/local/bin/ocx"))
    });
    assert_eq!(missing_adapter.availability, AcpAvailabilityStatus::AdapterMissing);
    assert!(missing_adapter.command.is_none());
```

- [ ] **Step 2: Run the complete preset test module**

Run:

```powershell
cargo test --manifest-path desktop/src-tauri/Cargo.toml discovery::presets::tests --lib
```

Expected: PASS.

- [ ] **Step 3: Format and check the edited Rust sources**

Run:

```powershell
cargo fmt --check
git diff --check
```

Expected: both commands exit 0.

- [ ] **Step 4: Commit verification additions**

```powershell
git add desktop/src-tauri/src/managed_agents/discovery/presets.rs
git commit -m "test: cover OCX harness availability"
```

## Explicit non-goals

- Do not read or write `~/.opencodex/config.json`.
- Do not start, stop, install, or update the OCX service from Buzz.
- Do not store an OpenRouter API key in Buzz.
- Do not add a new UI workflow; the existing harness catalog and per-agent environment overrides are sufficient.

## Task 4: Load OCX models directly in the existing picker

**Files:**

- Modify: `desktop/src-tauri/src/commands/agent_models.rs`
- Modify: `desktop/src-tauri/src/commands/agent_models_tests.rs`

- [x] Add a loopback OCX gateway catalog test that asserts the Anthropic model
  endpoint receives `limit=1000`, `ids=cli`, and the configured admission token.
- [x] Query that endpoint before the generic ACP subprocess in both saved-agent
  and draft-agent discovery. Require the OCX gateway-discovery flag and a
  loopback `ANTHROPIC_BASE_URL`, preserving user-supplied OCX admission tokens.
- [x] Bound the direct local request to three seconds and return the endpoint's
  `id` and `display_name` to Buzz's existing model picker.
- [x] Verify with the focused OCX model-discovery test.

## Self-review

- Scope coverage: the plan adds the OCX catalog entry, enforces both required executables, provides OCX gateway environment defaults, preserves user override precedence, and tests available/missing dependency states.
- Placeholder scan: no implementation or test step relies on deferred or undefined behavior.
- Type consistency: `PresetHarness.env` feeds both `AcpRuntimeCatalogEntry.definition_env` and `HarnessDefinition.env`; `required_commands` is used only by `preset_catalog_entry` availability resolution.
