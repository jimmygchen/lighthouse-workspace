# Claude Code Skills

Custom slash commands for Lighthouse development.

## Available Commands

| Command | Description |
|---------|-------------|
| `/review` | Code review with Lighthouse-specific safety checks |
| `/issue` | Create well-structured GitHub issues |
| `/release` | Generate release notes and announcements |

## Usage

```
/review
<paste code or PR link>
```

```
/issue
<describe the problem or feature>
```

```
/release
Generate release notes for v8.2.0
Base: stable
Release: release-v8.2
```

## Customization

Edit the `.md` files in this directory to adjust workflows. Claude Code automatically discovers and loads them as slash commands.
