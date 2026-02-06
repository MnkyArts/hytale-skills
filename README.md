# Hytale Agent Skills

> Please read the Disclaimer!

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

These skills are designed to be loaded by AI coding assistants that support the Agent Skills format. Point your assistant to the relevant `SKILL.md` file when working on that aspect of plugin development.

### Skills.sh Compatible

This repository is structured for compatibility with the [skills.sh](https://skills.sh) platform, which enables AI assistants to automatically discover and use these skills. Each skill includes:

- **YAML Front Matter**: Contains metadata (name, description, author, version)
- **Trigger Phrases**: Description includes quoted phrases that help AI assistants understand when to use the skill
- **Comprehensive Documentation**: Complete examples, patterns, and references
- **Reference Files**: Detailed technical documentation in `references/` subdirectories

No additional configuration files are required. AI assistants can consume these skills directly from the repository structure.

## Contributing

Found an error or want to improve a skill? Contributions are welcome. Keep in mind the experimental nature of this project.

When adding or modifying skills:
- Ensure each `SKILL.md` has complete YAML front matter
- Include trigger phrases in the description (quoted phrases like "create a command")
- Add comprehensive examples and patterns
- Update this README's skill list if adding new skills

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
