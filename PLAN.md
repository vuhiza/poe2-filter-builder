# Scalper PoE2 — In-Game Filter Overlay

## Context

PoE2 loot filters are plain-text `.filter` files that control item visibility and appearance. Editing them currently means alt-tabbing out. The user wants a **Windows overlay** to edit their NeverSink filter in-game — changing tiers, themes, sounds, colors — without leaving the game. PoE2 **auto-detects file changes**, so we just write the file. No price checking needed (user has Exiled Exchange 2).

---

## Tech Stack

| Layer | Choice | Why |
|-------|--------|-----|
| Framework | **Tauri v2** | ~10MB binary, ~30MB RAM. Rust backend fits the filter parser. |
| Backend | **Rust** | Strong types for filter DSL, atomic file writes. |
| Frontend | **Vue 3 + TypeScript + Pinia** | Proven for overlay UIs. |
| Hotkeys | **tauri-plugin-global-shortcut** | Windows global hotkey capture. |

No keyboard simulation needed — PoE2 auto-reloads on file change.

---

## PoE2 Filter Format

- `.filter` files in `Documents\My Games\Path of Exile 2\`
- Block-based: `Show`/`Hide`/`Minimal`/`Continue` with conditions + actions
- Conditions: `BaseType`, `Class`, `Rarity`, `ItemLevel`, `Sockets`, etc.
- Actions: `SetTextColor R G B A`, `SetBackgroundColor`, `SetBorderColor`, `SetFontSize`, `PlayAlertSound`, `CustomAlertSound`, `PlayEffect`, `MinimapIcon`, `DisableDropSound`
- First matching block wins (top-to-bottom)
- NeverSink uses FilterBlade metadata tags in comments: `# $type->currency $tier->t1`

---

## Project Structure

```
scalper-poe2/
├── src-tauri/
│   ├── Cargo.toml
│   └── src/
│       ├── main.rs
│       ├── commands/                 # Tauri IPC handlers
│       │   ├── filter_commands.rs    # parse, write
│       │   ├── theme_commands.rs     # list, apply, save
│       │   └── tier_commands.rs      # reassign, bulk ops
│       ├── domain/                   # Pure domain logic
│       │   ├── filter/
│       │   │   ├── block.rs          # FilterBlock, BlockType
│       │   │   ├── condition.rs      # Condition enum
│       │   │   ├── action.rs         # Action enum
│       │   │   ├── parser.rs         # .filter text → domain model
│       │   │   ├── serializer.rs     # domain model → .filter text
│       │   │   ├── filter.rs         # Filter aggregate root
│       │   │   └── metadata.rs       # NeverSink/FilterBlade tag parsing
│       │   ├── tier/
│       │   │   ├── tier.rs           # TierDefinition (S–E + Hidden)
│       │   │   └── assignment.rs     # Item-to-tier mappings
│       │   └── theme/
│       │       ├── theme.rs          # Theme aggregate
│       │       └── tier_style.rs     # Colors, sounds per tier
│       ├── application/              # Use cases
│       │   ├── ports/
│       │   │   ├── filter_repository.rs
│       │   │   └── config_repository.rs
│       │   ├── reassign_tier.rs
│       │   ├── apply_theme.rs
│       │   ├── toggle_visibility.rs
│       │   ├── edit_block_style.rs
│       │   └── import_filter.rs
│       └── infrastructure/
│           ├── filter_file_repo.rs   # .filter read/write (atomic)
│           ├── json_config_repo.rs   # JSON config persistence
│           └── path_detector.rs      # Find PoE2 filter directory
│
├── src/                              # Vue 3 frontend
│   ├── main.ts
│   ├── App.vue
│   ├── types/
│   ├── stores/
│   │   ├── filterStore.ts
│   │   ├── tierStore.ts
│   │   └── themeStore.ts
│   ├── composables/
│   │   ├── useOverlay.ts
│   │   └── useFilter.ts
│   ├── components/
│   │   ├── overlay/
│   │   │   ├── OverlayRoot.vue
│   │   │   ├── TierPanel.vue
│   │   │   ├── ThemeSelector.vue
│   │   │   ├── BlockEditor.vue
│   │   │   └── QuickActions.vue
│   │   └── settings/
│   │       ├── SettingsRoot.vue
│   │       ├── FilterImport.vue
│   │       ├── ThemeEditor.vue
│   │       └── HotkeyConfig.vue
│   └── ipc/
│
├── fixtures/                         # Test filter files
├── package.json
├── tauri.conf.json
└── CLAUDE.md
```

---

## Domain Model

### Filter (Aggregate Root)
```
Filter { path, blocks: Vec<FilterBlock>, preamble_comments }

FilterBlock {
  block_type: Show | Hide | Minimal,
  has_continue: bool,
  conditions: Vec<Condition>,
  actions: Vec<Action>,
  metadata: Option<BlockMetadata>,   # Parsed from NeverSink/FilterBlade tags
  raw_lines: Vec<String>,            # Preserved for round-trip fidelity
}

BlockMetadata { item_type, tier, group, tags: HashMap }
```

**Critical invariant**: round-trip fidelity. Unmodified blocks serialize identically to original text. Only changed action lines are regenerated.

### Tier System
```
TierDefinition { id: S|A|B|C|D|E|Hidden, label, default_visibility, style }
TierAssignment { block_index, tier_id, source: FromMetadata | UserOverride }
```

Tiers detected from NeverSink's `$tier->t1` metadata tags.

### Theme System
```
Theme { id, name, tier_styles: HashMap<TierId, TierStyle> }
TierStyle { text_color, bg_color, border_color, font_size, sound, minimap_icon, beam_effect }
```

---

## Core Data Flows

### Reassign Item Tier
```
Hotkey → overlay interactive → TierPanel shows items by tier
→ User moves "Chance Orb" from C → A
→ Tauri invoke: reassign_tier("Chance Orb", "A")
→ Rust: find matching blocks (metadata or BaseType)
→ Update block_type per tier default, apply theme style
→ Write .filter atomically (temp → rename)
→ PoE2 auto-detects change, filter is live
```

### Switch Theme
```
ThemeSelector → apply_theme("crimson")
→ For each tier-assigned block: replace color/sound/icon actions
→ Write .filter → PoE2 auto-reloads
```

---

## Config & Persistence

`%APPDATA%\scalper-poe2\`:
```
config.json              # filter path, hotkeys, active theme
tier_assignments.json    # user tier overrides (survives filter reimport)
themes/                  # one JSON per theme
backups/                 # auto-backup before each modification
```

Auto-detect filter path by scanning `Documents\My Games\Path of Exile 2\`.

---

## MVP Scope (v0.1)

1. Filter parser — Show/Hide blocks, all conditions/actions, round-trip fidelity
2. NeverSink metadata tag parsing (`$tier->`, `$type->`)
3. Overlay window — transparent, always-on-top, toggled by hotkey
4. Block list grouped by tier
5. Toggle block visibility (Show ↔ Hide)
6. Edit block colors, font size, alert sound
7. Atomic .filter write (PoE2 auto-reloads)
8. Auto-detect filter path on Windows
9. Config persistence

## v0.2 — Tier Management
- Tier panel with drag-and-drop reassignment
- Bulk tier ops (hide all D+, strictness presets)

## v0.3 — Theme System
- Theme definitions, switching, editing, import/export

## v0.4 — Polish
- Search/filter blocks, undo/redo, filter diffs, multiple profiles

---

## Testing Strategy

TDD, 95% minimum coverage.

- **Rust unit**: every domain method (parsing, serialization, tier logic)
- **Rust property**: parse→serialize round-trip on generated filters
- **Rust integration**: real NeverSink `.filter` fixture files
- **Vue**: component + store tests with Vitest, IPC mocked
- **E2E**: import filter → modify → verify output file

---

## Verification

1. `cargo test` — all pass
2. `npm run test` — all pass
3. Manual: load NeverSink filter, toggle visibility, confirm PoE2 picks up change
4. Manual: switch theme, verify colors in .filter file
5. Coverage ≥ 95%

---

## Implementation Order

1. `src-tauri/src/domain/filter/parser.rs` — foundation, everything depends on it
2. `src-tauri/src/domain/filter/block.rs` — core data structure
3. `src-tauri/src/domain/filter/serializer.rs` — round-trip serialization
4. `src-tauri/src/domain/filter/metadata.rs` — NeverSink tag parsing
5. `src-tauri/src/infrastructure/filter_file_repo.rs` — atomic file I/O
6. `src-tauri/src/commands/filter_commands.rs` — IPC bridge
7. `src/components/overlay/OverlayRoot.vue` — overlay shell
8. `src/components/overlay/TierPanel.vue` — main interaction surface
