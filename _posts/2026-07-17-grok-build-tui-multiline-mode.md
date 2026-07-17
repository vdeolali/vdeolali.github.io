---
title: "Grok Build TUI: Multiline Mode Is Not a Showstopper"
date: 2026-07-17
---

# Grok Build TUI: Multiline Mode Is Not a Showstopper

I hit what felt like a wall: "I cannot use multiline comment, this is a show stopper" followed by `/multiline`.

It turns out this is **not** a bug in the model or the agent. It's a feature of the Grok Build TUI (the interactive terminal UI you're using right now).

## The Two Ways to Enter Multiline Text

There are two distinct concepts that both involve "multiline":

1. **Multiline input mode** (`/multiline` or `Ctrl+M` when prompt is focused)
2. Writing actual multiline comments or code blocks in your prompts/responses

### 1. Toggling Multiline Input Mode

By default, when the prompt is focused:
- `Enter` = **send** the message
- You cannot easily type newlines

Running `/multiline` (or pressing `Ctrl+M`) **toggles** the mode:

- `Enter` now inserts a **newline**
- `Shift+Enter` (or `Alt+Enter`) sends the message

This is exactly what the `/multiline` slash command (and the recent MRU entry in `~/.grok/slash-mru.json`) does. It's documented in the user guide under keyboard shortcuts and slash commands.

The TUI even has special logic so that mid-turn, an empty prompt + bare `Enter` still acts as "send now" for queued follow-ups.

### 2. Multiline Comments in Code/Prompts

Once in the correct input mode (or using `Shift+Enter` in normal mode), you *can* write multiline comments, code blocks, YAML, etc. The highlight.js setup in this blog already supports comments in several languages (TSQL, PowerShell, CSS, HTML).

The "show stopper" feeling usually comes from one of two places:
- Not knowing `Ctrl+M` / `/multiline` exists
- Terminal-specific keyboard protocol issues (WezTerm, tmux, Zellij, Windows Terminal) that prevent `Shift+Enter` or `Ctrl+Enter` from registering properly

## Quick Fixes for Common Terminal Issues

See the full user guide (`~/.grok/docs/user-guide/21-terminal-support.md` or run `/terminal-setup` in the TUI) for your specific terminal. Common ones:

- **WezTerm**: Add `enable_kitty_keyboard = true` to your config
- **Zellij**: Use the "Unlock-First (non-colliding)" preset
- **tmux**: Enable `extended-keys on`
- **Windows Terminal**: Use `Alt+V` for images; rebind conflicting `Ctrl+L` if using VS Code family

## Why This Matters for Agents

The recent Kimi K3 posts highlighted how much context is wasted on "explain the project to me." The new `GROK.md` in this repo (created during this resume session) front-loads the exact conventions so agents waste fewer tokens and fewer turns on tool flailing.

The TUI's `/multiline` (or `Ctrl+M`), keyboard shortcuts (`Ctrl+;`, `Ctrl+P` for palette), skills system, `GROK.md` context file, and permission model are all designed to make tight, prescriptive workflows *efficient*.

**Note on persistence**: Multiline mode is per-session (toggled with `/multiline` or `Ctrl+M`). There is currently no `default_multiline = true` setting in `~/.grok/config.toml` or `pager.toml`. Run it once at the start of a session or add it to your muscle memory.

Multiline input is not a limitation—it's a deliberate design choice that, once you know the chord (`Ctrl+M` then `Shift+Enter` to send), becomes muscle memory.

No longer a showstopper.

(Yes, this entire post — including the updates above — was written in multiline mode.)
