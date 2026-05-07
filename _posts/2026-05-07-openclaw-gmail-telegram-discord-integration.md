---
title: "Integrating OpenClaw with Gmail, Telegram, and Discord"
date: 2026-05-07
---

# Integrating OpenClaw with Gmail, Telegram, and Discord

I wrote a shorter post earlier on my old blog, [OpenClaw is MicroSaaS](https://www.networkofthings.com/2026_04_01_archive.html#4651470299340435740), after I got OpenClaw working against my Gmail. That post was more about the experience. This one is the technical version: what OpenClaw is actually doing under the hood, why the integration takes real setup effort, and how I wired it into **Gmail**, **Telegram**, and **Discord** from a local machine.

The short version is this: OpenClaw is not just a chat bot with a pretty shell. It is a **local-first gateway** that owns messaging surfaces, tools, sessions, and automation. Once I started looking at it that way, the setup made much more sense.

## What OpenClaw actually is

The upstream project is [openclaw/openclaw](https://github.com/openclaw/openclaw), and the architecture is centered around a long-lived local **Gateway**. The Gateway exposes a WebSocket control plane, maintains channel connections, and routes events into agent sessions.

That architecture matters because the three integrations I set up do not all enter the system the same way:

- **Telegram** is a native messaging channel connected directly to the OpenClaw gateway.
- **Discord** is also a native channel, but it needs a Discord application, bot token, gateway intents, and then pairing or allowlist configuration inside OpenClaw.
- **Gmail** is different. It is not just "another chat channel". In practice it becomes an **event-driven integration** using `gog`, Google OAuth, Gmail watch, Google Pub/Sub, and OpenClaw hooks.

That is why the setup feels long. You are really stitching together several trust boundaries:

1. Your model provider auth for the LLM itself.
2. Your local OpenClaw gateway runtime.
3. Google OAuth plus Pub/Sub for Gmail.
4. Bot credentials and access control for Telegram and Discord.

## My setup assumptions

I did this on **WSL2**, which OpenClaw explicitly recommends for Windows users. The upstream docs currently recommend **Node 24** or at least **Node 22.16+**.

At a minimum, I would have these pieces ready before starting:

- `node` and `npm`
- `openclaw`
- `gcloud`
- `gog`
- a model account or API key for the LLM provider I want OpenClaw to use
- a Telegram bot token from `@BotFather`
- a Discord bot token from the Discord Developer Portal

The recommended install path is:

```bash
npm install -g openclaw@latest
openclaw onboard --install-daemon
```

If I want to run it in the foreground while debugging:

```bash
openclaw gateway --port 18789 --verbose
```

The important mental model is that the gateway is the control plane. Once it is up, the channels and automations hang off of it.

## Why Gmail is the hardest integration

Telegram and Discord are basically messaging front ends. Gmail is not.

For Gmail, I had to solve three things:

- OAuth access to the mailbox
- a way for Gmail to notify my local OpenClaw stack when new mail lands
- a safe handoff path from that notification into an OpenClaw agent session

OpenClaw's docs handle this with **Gmail Pub/Sub integration** and the `gog` CLI.

### Step 1: authorize `gog` for Google services

OpenClaw uses `gog` for Google Workspace access. The setup is:

```bash
gog auth credentials /path/to/client_secret.json
gog auth add you@gmail.com --services gmail,calendar,drive,contacts,docs,sheets
gog auth list
```

That gives `gog` the OAuth credentials it needs to access Gmail and related Google APIs.

### Step 2: enable Gmail API and Pub/Sub

The OpenClaw docs use Google Cloud Pub/Sub as the event path for Gmail watch notifications:

```bash
gcloud auth login
gcloud config set project <project-id>
gcloud services enable gmail.googleapis.com pubsub.googleapis.com
```

Then create a topic and grant Gmail permission to publish into it:

```bash
gcloud pubsub topics create gog-gmail-watch
gcloud pubsub topics add-iam-policy-binding gog-gmail-watch \
  --member=serviceAccount:gmail-api-push@system.gserviceaccount.com \
  --role=roles/pubsub.publisher
```

### Step 3: start the Gmail watch

The lower-level manual path is:

```bash
gog gmail watch start \
  --account you@gmail.com \
  --label INBOX \
  --topic projects/<project-id>/topics/gog-gmail-watch
```

The OpenClaw-friendly path is simpler:

```bash
openclaw webhooks gmail setup --account you@gmail.com
```

That command writes `hooks.gmail` configuration, enables the Gmail preset, and prepares the push endpoint that the gateway will use.

### Step 4: let the gateway own the watcher

This is the part that made the architecture click for me. Once `hooks.enabled=true` and `hooks.gmail.account` is configured, the **gateway starts `gog gmail watch serve` on boot and auto-renews the watch**.

So the runtime flow becomes:

1. Gmail sees a mailbox event.
2. Gmail publishes the event to the Pub/Sub topic.
3. `gog` receives and normalizes that watch event.
4. OpenClaw hooks ingest it.
5. The gateway routes the event into an agent session.

That is a very different pattern from a bot reading chat messages, and it explains why Gmail setup feels more like building an automation pipeline than enabling a simple inbox plugin.

## Telegram integration

Telegram was much more straightforward.

### Step 1: create a bot in BotFather

In Telegram, I created a bot with `@BotFather`, ran `/newbot`, and saved the token.

### Step 2: configure the channel in OpenClaw

OpenClaw's Telegram configuration can be as simple as:

```json5
{
  channels: {
    telegram: {
      enabled: true,
      botToken: "123:abc",
      dmPolicy: "pairing",
      groups: { "*": { requireMention: true } },
    },
  },
}
```

Telegram does not use a separate `openclaw channels login telegram` flow. I just configure the token and start the gateway.

### Step 3: approve the first DM

With `dmPolicy: "pairing"`, the first DM from an unknown sender gets a code instead of being processed immediately.

```bash
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

That pairing step is one of the better design choices in OpenClaw. It keeps my personal assistant from turning into a public endpoint just because someone discovers the bot username.

### Step 4: optional group behavior

If I want the bot in a group, I can add the bot to the group and then decide whether the group needs explicit mention or always-on behavior. Telegram also has its own privacy-mode behavior, so group visibility depends partly on BotFather settings and partly on OpenClaw config.

For a private group, I would usually stay restrictive first and then loosen it only if I really want ambient participation.

## Discord integration

Discord was somewhere in between Telegram and Gmail. It is not as operationally heavy as Gmail, but it is definitely more involved than Telegram because Discord wants a properly configured application and bot.

### Step 1: create the application and bot

In the Discord Developer Portal, I created a new application, added a bot, and copied the bot token.

The critical part is enabling the right gateway intents:

- **Message Content Intent** is required.
- **Server Members Intent** is recommended.
- **Presence Intent** is optional.

### Step 2: invite the bot with usable permissions

From the OAuth2 URL Generator, I enabled:

- `bot`
- `applications.commands`

And I made sure the bot had at least:

- View Channels
- Send Messages
- Read Message History
- Embed Links
- Attach Files

If I wanted thread-heavy usage, I would also enable **Send Messages in Threads**.

### Step 3: configure the token in OpenClaw

I prefer not to store bot tokens inline, so an environment-backed config is cleaner:

```bash
export DISCORD_BOT_TOKEN="YOUR_BOT_TOKEN"
```

```json5
{
  channels: {
    discord: {
      enabled: true,
      token: { source: "env", provider: "default", id: "DISCORD_BOT_TOKEN" },
    },
  },
}
```

Then I can start the gateway and pair from Discord DM.

### Step 4: approve the first DM pairing

Once the gateway is running, I DM the bot in Discord and approve the pairing code:

```bash
openclaw pairing list discord
openclaw pairing approve discord <CODE>
```

At that point Discord DMs behave like a controlled front end into my assistant.

### Step 5: make a private server usable

For a private guild, I can explicitly allow my server and user ID:

```json5
{
  channels: {
    discord: {
      groupPolicy: "allowlist",
      guilds: {
        YOUR_SERVER_ID: {
          requireMention: true,
          users: ["YOUR_USER_ID"],
        },
      },
    },
  },
}
```

That is a good default because it keeps the bot scoped to a server I control instead of wandering across whatever guilds it happens to be invited into.

## The integration pattern that finally made sense to me

Once I had all three running, the OpenClaw model became much clearer:

- **Telegram** and **Discord** are conversational ingress channels.
- **Gmail** is an automation ingress path.
- The **Gateway** is the stable center of the system.

That is the part I missed the first time around. I originally thought of OpenClaw as "an AI that can check my Gmail". Technically, it is closer to a **personal agent gateway** with multiple inbound surfaces and a local execution/control plane.

That explains the setup complexity, but it also explains the power. Once the integrations are in place, I can ask questions across those surfaces in a way that a conventional email client or single-channel bot does not really support.

For example, I can use Gmail as the data source, then talk to the assistant through Telegram or Discord, and let the same gateway manage the session, tooling, and response path.

## Security notes that matter

This stack touches real personal communications, so I would not run it casually without guardrails.

The OpenClaw docs are pretty explicit about a few things that I agree with:

- keep DM policy on **pairing** or explicit allowlists
- keep Gmail hooks behind loopback, tailnet, or a trusted reverse proxy
- use a dedicated hook token instead of reusing the gateway auth token
- keep the bot in private Telegram groups or private Discord servers unless there is a real reason not to

The right default mindset is that inbound messages are **untrusted input**.

## What I would tell someone before they try this

If someone wants to integrate OpenClaw with Gmail, Telegram, and Discord, this is what I would tell them up front:

- **Telegram is the easiest**. Get a bot token, start the gateway, approve pairing.
- **Discord is manageable**, but you need to be careful with intents, permissions, and allowlists.
- **Gmail is the most technical** because it is really a Google OAuth plus Pub/Sub plus webhook pipeline.
- **WSL2 is a good place to do this on Windows** because that is the path OpenClaw itself recommends.
- **The local gateway is the whole point**. If that architectural model does not make sense to you yet, the rest of the setup will feel random and frustrating.

## Closing thought

What impressed me most about OpenClaw was not that it could read my email. Plenty of tools can do that. What impressed me was that it gave me a way to unify **personal messaging channels**, **automation hooks**, and **agent execution** around a local gateway that I control.

That is also why the setup is not trivial. The project is doing real systems integration work, not just wrapping an LLM in a chat window.
