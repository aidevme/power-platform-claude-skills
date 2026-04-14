# Power Platform Skills for Claude Code

![GitHub release](https://img.shields.io/github/v/release/aidevme/power-platform-claude-skills)
![GitHub license](https://img.shields.io/github/license/aidevme/power-platform-claude-skills)
![Automated Version Increment and Tagging](https://github.com/aidevme/power-platform-claude-skills/workflows/Automated%20Version%20Increment%20and%20Tagging/badge.svg)
![Power Platform](https://img.shields.io/badge/Power%20Platform-Dataverse-742774)
![Claude Skills](https://img.shields.io/badge/Claude-Skills-orange)

A domain skill that gives [Claude Code](https://claude.ai/claude-code) deep knowledge of unit testing Microsoft Dataverse plugins with the FakeXrmEasy framework.

## Skill

| Skill | What It Does |
|---|---|
| **dataverse-plugins-unit-testing** | Unit testing Dataverse plugins with FakeXrmEasy framework — covers test setup, in-memory context, pipeline simulation, entity images, mocking, and assertion patterns |

## Installation

1. Copy the skill folder into your project:
   ```bash
   git clone https://github.com/aidevme/power-platform-claude-skills.git
   mkdir -p .claude/skills
   cp -r power-platform-claude-skills/dataverse-plugins-unit-testing .claude/skills/
   ```

2. Add a `CLAUDE.md` to your project root that references the skill.

3. Start Claude Code in your project — the skill loads automatically.

### Installing a Specific Version

To use a specific version of the skill, clone the repository at a particular tag:

```bash
# List available versions
git ls-remote --tags https://github.com/aidevme/power-platform-claude-skills.git

# Clone specific version
git clone --branch v1.0.0 https://github.com/aidevme/power-platform-claude-skills.git

# Or checkout a specific version in existing clone
cd power-platform-claude-skills
git checkout v1.0.0
```

See [Releases](https://github.com/aidevme/power-platform-claude-skills/releases) for all available versions.

## Versioning

This project follows [Semantic Versioning](https://semver.org/):
- **Major** (x.0.0) — Breaking changes or major new capabilities
- **Minor** (0.x.0) — New features, backward compatible
- **Patch** (0.0.x) — Bug fixes, backward compatible

Version bumps are automated via GitHub Actions when commits include:
- `[major]` — Bump major version
- `[minor]` — Bump minor version  
- `[patch]` — Bump patch version (default)

See [CHANGELOG.md](CHANGELOG.md) for version history.

## Samples

Working examples and sample code demonstrating the dataverse-plugins-unit-testing skill in action are available in the samples repository:

**Repository:** [power-platform-claude-skills-samples](https://github.com/aidevme/power-platform-claude-skills-samples)

The samples repo includes:
- Complete plugin projects with unit tests
- Real-world testing scenarios
- Best practices examples
- Integration with CI/CD pipelines

```bash
git clone https://github.com/aidevme/power-platform-claude-skills-samples.git