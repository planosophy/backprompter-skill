# Backprompter skill for Claude

A Claude skill that teaches Claude
how to build apps on [Backprompter](https://backprompter.com). With it installed, you can
ask Claude Code to write an agent definition, wire your frontend to the browser SDK, or
call your agents from a server, and it will get the fields, calls and limits right.

What it covers:

- Agent JSON you can import in the builder, with every field explained
- Retrieval (RAG) sources, combo agents, API tools and MCP servers
- Choosing a model, and the traps with reasoning models
- The browser SDK, the server SDK, and keeping your own login
- Calling your agents from MCP clients
- Evaluations, team policies, embedding the chat, and payments

## Install in Claude Code

```
/plugin marketplace add planosophy/backprompter-skill
/plugin install backprompter@planosophy
```

Claude then uses the skill whenever you work on a Backprompter app. Run
`/plugin marketplace update planosophy` to pick up new versions.

### Or copy the file

The skill is one file. To add it to a single project without the plugin, save it as
`.claude/skills/backprompter/SKILL.md` in that project:

```bash
mkdir -p .claude/skills/backprompter
curl -L https://raw.githubusercontent.com/planosophy/backprompter-skill/main/plugins/backprompter/skills/backprompter/SKILL.md \
  -o .claude/skills/backprompter/SKILL.md
```

Use `~/.claude/skills/backprompter/` instead to make it available in every project.

## Using it elsewhere

The file is plain Markdown. Any assistant that accepts instructions or project knowledge,
such as a Claude Project, can use it as a reference for Backprompter.

## Related

- [Browser SDK docs](https://github.com/planosophy/backprompter-sdk-docs)
- [Server SDK docs](https://github.com/planosophy/backprompter-server-sdk-docs)
- [AI docs assistant](https://github.com/planosophy/ai-docs-assistant), an example app

## License

MIT
