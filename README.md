# Schemity skills

Agent skills for designing relational database schemas, from the team behind [Schemity](https://schemity.com), database design software that reads your live database and shows the impact of every change before it runs.

## designing-database-schemas

Makes an AI coding agent decide what it would otherwise leave to the database's defaults:

- a name on every constraint and index
- an `ON DELETE` on every foreign key, with the reason
- money as `numeric`, never a float
- the right identity and name for link and membership tables
- the locking cost of every statement in a migration on a table that holds data
- a verification step before it hands the design back

[`reference.md`](skills/designing-database-schemas/reference.md) holds the rest of the defaults (keys, time, text, enums, unique rules, deferrable and circular foreign keys, polymorphic associations, embeddings), one line each, linked to the post with the measurements behind it.

Works on its own. With the [Schemity MCP server](https://schemity.com/doc/ai-assisted-design/) connected, the agent also stages the design on your diagram and checks it with Schemity's lint and impact analysis.

### Install

Claude Code:

```sh
git clone https://github.com/schemity/skills.git
cp -r skills/skills/designing-database-schemas ~/.claude/skills/
```

Other agents that read [Agent Skills](https://agentskills.io): copy the folder into `~/.agents/skills/`.

## License

MIT
