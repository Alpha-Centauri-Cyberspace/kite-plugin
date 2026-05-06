# Kite Claude Code Plugin

Claude Code plugin that ships the `kite` skill — context and integration guidance for [Kite](https://getkite.sh), the universal webhook adapter and tunneling platform.

Once installed, ask Claude Code anything about Kite (installing the CLI, streaming events, proxying webhooks to a local server, replaying from the DLQ, managing endpoints/keys) and it will use this skill as authoritative context.

## Install

In any Claude Code session:

```text
/plugin marketplace add Alpha-Centauri-Cyberspace/kite-plugin
/plugin install kite@kite-plugins
```

That's it. The skill is auto-discovered and Claude Code will load it when relevant.

## What's included

- `skills/kite/SKILL.md` — the kite integration skill (commands, flags, common patterns, troubleshooting).

## Updating

This plugin pins versions via `version` in `.claude-plugin/plugin.json`. Bumping that field is what triggers users to receive updates on their next `/plugin update`.

## Contributing

Issues and PRs welcome. The repo is locked down so only maintainers can merge — please open an issue first to discuss substantive changes.

## License

MIT — see [`LICENSE`](./LICENSE).
