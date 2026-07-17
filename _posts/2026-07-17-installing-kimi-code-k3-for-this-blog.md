---
title: "Installing Kimi Code, Funding It, and Using Kimi K3 to Write This Blog"
date: 2026-07-17
---

# Installing Kimi Code, Funding It, and Using Kimi K3 to Write This Blog

This post is a little meta: I used Kimi Code to help write the post about installing Kimi Code.

I had been using Claude-style terminal workflows for small projects like this blog, but I wanted to try Kimi Code's coding endpoint with Kimi K3. The goal was not to replace every tool I use. The goal was to see whether I could install Kimi Code, get the right API key, add some credit, and then point my existing terminal coding workflow at Kimi instead of Anthropic.

The short version: it worked, and the main thing to understand is that there are two related but different Kimi API paths.

## Kimi Code vs Kimi Platform

The first thing I had to get straight was the difference between **Kimi Code** and the broader **Kimi Platform**.

For this blog workflow, I cared about **Kimi Code**, because it is meant for terminal and IDE coding agents. The Kimi Code docs describe it as an intelligent programming service built on Kimi's flagship models, available through the CLI, VS Code, and third-party coding tools.

That distinction matters because the API endpoints are different:

- Kimi Code Anthropic-compatible base URL: `https://api.kimi.com/coding/`
- Kimi Code OpenAI-compatible base URL: `https://api.kimi.com/coding/v1`
- Kimi Open Platform base URL: `https://api.moonshot.cn/v1`

For Claude Code-style integration, the important one is the Anthropic-compatible Kimi Code endpoint.

## Installing Kimi Code CLI

On Linux, the install was simple:

```bash
curl -fsSL https://code.kimi.com/kimi-code/install.sh | bash
```

The installer downloads the latest release, verifies the checksum, and puts the `kimi` executable on the `PATH`.

After installation, I checked that the CLI was available:

```bash
kimi --version
```

The CLI can also be installed through npm, but that path requires Node.js `22.19.0` or later:

```bash
npm install -g @moonshot-ai/kimi-code
```

I used the install script because it is the recommended path and does not require me to manage Node first.

## First launch and login

The normal first-run flow is:

```bash
cd your-project
kimi
```

Then inside the interactive UI:

```text
/login
```

The login flow supports two options:

- **Kimi Code (OAuth)** — device-code flow
- **Kimi Platform API key** — for the separate Kimi Platform

For my Claude Code setup, I did not rely on the OAuth flow inside `kimi`. I needed an API key I could export as an environment variable.

## Getting the API key

The API key for Kimi Code comes from the Kimi Code Console. The docs say members can create and manage up to five API keys, and each key is shown only once when created.

That part is important: once the dialog is closed, the full key cannot be viewed again. So the workflow is:

1. Open the Kimi Code Console.
2. Create a new API key.
3. Copy it immediately.
4. Store it somewhere safe.
5. Never paste it into a blog post, screenshot, repo, or commit.

For the actual setup, the key goes into an environment variable, not into source control.

## Funding the account

Before pointing a coding agent at the endpoint, I wanted to make sure the account had a usable balance path. I funded the setup with about **$50** so I would not stop in the middle of a longer coding session.

The docs separate subscription quota from **Extra Usage**. Subscription quota is consumed first; Extra Usage is the fallback balance when quota runs out. The exact billing display and limits are shown in the Kimi subscription/console pages, but the operational model is simple:

- keep the membership/subscription active
- enable Extra Usage if you want tasks to continue after quota is exhausted
- set a monthly spending cap if you want a hard limit

For a coding-agent workflow, that fallback matters. A long refactor or multi-file edit is exactly the kind of task I do not want dying halfway through because a quota window ran out.

## The Claude Code part: pointing Claude at Kimi K3

This was the part I actually cared about most.

I did not want to give up the Claude Code terminal flow immediately. I wanted to keep the same style of interaction but have the backend calls go to Kimi Code instead.

The Kimi docs show the Claude Code integration using environment variables. For Kimi K3 with the large context window, the example is:

```bash
export ANTHROPIC_BASE_URL=https://api.kimi.com/coding/
export ANTHROPIC_API_KEY="your-kimi-code-api-key"

export ANTHROPIC_MODEL="k3[1m]"
export ANTHROPIC_DEFAULT_FABLE_MODEL=$ANTHROPIC_MODEL
export ANTHROPIC_DEFAULT_OPUS_MODEL=$ANTHROPIC_MODEL
export ANTHROPIC_DEFAULT_SONNET_MODEL=$ANTHROPIC_MODEL
export ANTHROPIC_DEFAULT_HAIKU_MODEL=$ANTHROPIC_MODEL
export CLAUDE_CODE_SUBAGENT_MODEL=$ANTHROPIC_MODEL

export CLAUDE_CODE_AUTO_COMPACT_WINDOW=1048576
export CLAUDE_CODE_MAX_CONTEXT_TOKENS=1048576

claude
```

The exact model value depends on membership tier. The docs list the model IDs as:

- `k3` — Kimi K3
- `kimi-for-coding` — Kimi K2.7 Code
- `kimi-for-coding-highspeed` — Kimi K2.7 Code HighSpeed

They also note that `k3[1m]` is the 1M-context variant for higher tiers, while lower tiers may need to use `k3` or `kimi-for-coding` with the smaller context window.

After starting Claude Code with those variables set, the verification step is:

```text
/status
```

If the Base URL shows `https://api.kimi.com/coding/`, the configuration is pointed at Kimi. The docs also note that even if the UI still shows a Claude model name, the actual calls are going to the Kimi Code API.

## Using it on this blog

Once the endpoint was configured, the workflow became very ordinary, which is exactly what I wanted.

I went into the local blog repo:

```bash
cd ~/vdeolali.github.io
```

Then I asked the agent to inspect the blog structure and find where posts live. The important part is that the repo is just a normal Jekyll site:

```text
_posts/
_config.yml
index.md
archive.md
```

A new post is a new markdown file in `_posts` with a filename like:

```text
YYYY-MM-DD-slug.md
```

The front matter is minimal:

```yaml
---
title: "Post title"
date: YYYY-MM-DD
---
```

After that, the workflow is the same as any other small coding task:

- read the existing posts to match tone
- create the new markdown file
- verify the filename and front matter
- do not commit or push until I approve

That last rule matters. I want the agent to be useful, but I still want the final human checkpoint before anything becomes public.

## What I liked

The biggest thing I liked is that Kimi Code is not trying to replace every developer tool I already know. It can be used directly through the `kimi` CLI, but it can also sit behind tools that speak the Anthropic or OpenAI API shape.

For this experiment, that meant I could keep the Claude Code-style terminal flow while changing the model backend to Kimi K3.

I also liked that the model IDs are explicit. If I want K3, I can ask for `k3`. If I want the high-speed K2.7 coding model, that is a separate ID: `kimi-for-coding-highspeed`.

## What to watch out for

There are a few practical gotchas:

- Do not confuse Kimi Code with the Kimi Open Platform. They have different base URLs and billing models.
- Copy the API key immediately. The console says the full key is shown only once.
- Keep the key out of the repo and out of shell history if possible.
- Use `/status` in Claude Code to confirm the Base URL really is `https://api.kimi.com/coding/`.
- If you use the high-speed model ID, it has to be exact: `kimi-for-coding-highspeed`.
- If your plan does not include a model, do not assume the request failed loudly; check the docs for your tier and model access.

## Final thought

The setup was less mysterious than I expected. The real work was not the install command. It was understanding which Kimi product I was using, which endpoint belonged to that product, and how to wire that into the terminal workflow I already liked.

Once those pieces were clear, using Kimi K3 to work on this blog felt normal: read the repo, make the change, show me the diff, and wait for approval before publishing.

That is the workflow I wanted.
