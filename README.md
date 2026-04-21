# claude-plugins

Claude Code plugin marketplace by [RNViththagan](https://github.com/RNViththagan).

## Install

Add this marketplace to Claude Code once:

```
/plugin marketplace add RNViththagan/claude-plugins
```

Then install any plugin:

```
/plugin install <plugin-name>@RNViththagan
```

## Contributing

To add a new plugin:

1. Create `plugins/<plugin-name>/` with the structure:
   ```
   plugins/<plugin-name>/
   ├── .claude-plugin/
   │   └── plugin.json
   └── skills/
       └── <skill-name>/
           └── SKILL.md
   ```
2. Add an entry to `.claude-plugin/marketplace.json`
3. Open a PR
