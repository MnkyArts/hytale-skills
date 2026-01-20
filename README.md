# Hytale Plugin Development Skills

A collection of Agent Skills for AI-assisted Hytale server plugin development.

## What is this?

This repository contains structured skill files that help AI coding assistants (like Claude/OpenCode) understand and generate Hytale server plugin code. Each skill covers a specific domain of plugin development with patterns, examples, and reference documentation.

## Available Skills

| Skill | Description |
|-------|-------------|
| `hytale-plugin-basics` | Plugin structure, manifest, lifecycle, registries |
| `hytale-custom-blocks` | Custom block types, materials, states |
| `hytale-custom-items` | Weapons, armor, tools, item stats & effects |
| `hytale-custom-entities` | Entity types, ECS components, AI sensors & actions |
| `hytale-custom-assets` | Models, textures, sounds, animations |
| `hytale-events-api` | Event system, listeners, custom events |
| `hytale-networking` | Packets, serialization, client-server communication |
| `hytale-commands` | Custom commands, arguments, permissions |
| `hytale-ui-windows` | UI windows, containers, crafting interfaces |
| `hytale-crafting-recipes` | Crafting recipes, bench types, recipe registration |

## Disclaimer

**These skills are HEAVILY and FULLY VibeCoded.**

This means:
- All content was generated through AI-assisted analysis and pattern matching
- Information **COULD be wrong** or inaccurate
- Nothing is 100% tested against a live Hytale server
- API details may change as Hytale development continues
- Use at your own risk and verify critical implementations

The skills are meant as a starting point and reference, not as authoritative documentation. Always cross-reference with official Hytale modding resources when available.

## Structure

```
skills/
├── hytale-plugin-basics/
│   ├── SKILL.md              # Main skill instructions
│   └── references/           # Detailed reference docs
├── hytale-custom-blocks/
├── hytale-custom-items/
├── hytale-custom-entities/
├── hytale-custom-assets/
├── hytale-events-api/
├── hytale-networking/
├── hytale-commands/
├── hytale-ui-windows/
└── hytale-crafting-recipes/
```

## Usage

These skills are designed to be loaded by AI coding assistants that support the Agent Skills format. It Points your assistant to the relevant `SKILL.md` file when working on that aspect of plugin development.

## Contributing

Found an error or want to improve a skill? Contributions are welcome. Keep in mind the experimental nature of this project.

## License

MIT
