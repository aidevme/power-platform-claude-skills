# Power Platform Skills for Claude Code

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