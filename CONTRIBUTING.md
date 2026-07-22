# Contributing

Thank you for your interest in contributing to the Azure SRE Agent Skills Pack!

## How to contribute

### Adding a new skill

1. Create a new folder under `skills/` following the naming pattern: `XX-skill-name/`
2. Create a `SKILL.md` file with the required YAML front matter:
   ```yaml
   ---
   name: your-skill-name
   description: Clear description of what the skill does and when to use it. Include trigger keywords.
   tools:
     - RunAzCliReadCommands
     - execute_kusto_query
   ---
   ```
3. Follow the structure of existing skills (Purpose, When to use, Pre-check, Procedure, Expected output)
4. Test the skill in your own SRE Agent instance
5. Submit a PR with:
   - The SKILL.md file
   - Updated README.md (add to the skills table)
   - Description of what gap this skill fills

### Improving existing skills

- Fix az cli commands (verify they work in the SRE Agent sandbox)
- Add edge cases or additional checks
- Improve output formatting
- Add references to Microsoft documentation

### Guidelines

- **Read-only by default**: Skills should use `RunAzCliReadCommands`, not write commands
- **Test your commands**: Verify every az cli command works as written
- **Date handling**: Use `$(date -u -d '...' +%Y-%m-%dT%H:%M:%SZ)` for date calculations (Linux sandbox)
- **No hardcoded values**: Use `<placeholder>` syntax for user-specific values
- **Keep skills focused**: One skill = one use case. Don't create a "do everything" skill.
- **Respect the 5-skill limit**: Azure SRE Agent loads max 5 concurrent skills

### Code of Conduct

Be kind, be constructive, be helpful. We're all here to make Azure operations better.

## Reporting issues

Found a broken command or incorrect behavior? Open an issue with:
1. Which skill is affected
2. What you expected
3. What actually happened
4. Your Azure CLI version (`az version`)
