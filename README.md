# Ramly for AI assistants

Run **reliability, availability and maintainability (RAM) studies** from Claude,
ChatGPT, Microsoft Copilot, Cursor, VS Code and other AI tools with
[Ramly](https://ramly.io).

You describe a plant, process or fleet. The assistant:
- builds the reliability block diagram;
- runs Ramly's **discrete-event Monte Carlo** or **exact analytical (Markov)** engine on your Ramly account;
- explains availability, downtime, bad actors, spares, crews and costs;
- compares design options.

This repository contains:

| | |
|---|---|
| `skills/ramly-ram-analysis/` | An [Agent Skill](https://agentskills.io) with the RAM study workflow, modeling patterns, distribution guidance, result interpretation and worked examples |
| `.mcp.json` / `mcp.json` | The Ramly MCP server, `https://mcp.ramly.io/mcp` (OAuth sign-in to your Ramly account) |
| `.claude-plugin/` | Claude plugin manifest and marketplace |
| `plugin.json` | [Agent Plugins 1.0](https://agent-plugins.org) manifest (Codex, Copilot, Cursor and others) |

A Ramly account is required. Studies use your plan's monthly run credits. Plans are
at [ramly.io](https://ramly.io).

## Install

**Claude Code**
```
/plugin marketplace add arslanerdem/ramly-plugin
/plugin install ramly@ramly
```
The first time a Ramly tool runs, Claude opens a browser to sign in to Ramly.

**Claude (web, desktop, mobile)**
1. Go to Settings → Connectors → *Add custom connector*.
2. Enter the URL `https://mcp.ramly.io/mcp`.
3. To add the skill, upload `skills/ramly-ram-analysis` as a ZIP under Settings → Capabilities → Skills.

**ChatGPT, Microsoft Copilot Studio, VS Code, Cursor and other MCP clients:** add an
MCP server or connector with the URL `https://mcp.ramly.io/mcp` and sign in when asked.
Clients that support Agent Plugins or Agent Skills can install this repository directly.

## What you can ask

- "Our cooling water system has three pumps (2 needed) and a heat exchanger. Model it in Ramly and tell me the availability."
- "Would a second spare pump cartridge pay for itself?"
- "Which components drive our downtime? Show the importance ranking."
- "What PFD and SIL does our 1oo2 high-pressure trip reach with annual proof testing?"
- "Compare the duty/standby design with 2 × 100 % hot redundancy."

## Data and privacy

- The MCP server acts only on the Ramly account you connect. Models and results live in that account, and you can view them on ramly.io.
- Disconnect any time under Account → **Connected AI apps**.
- See the [privacy policy](https://ramly.io/privacy) and [terms](https://ramly.io/terms).

## Support

Open an issue in this repository or visit [ramly.io](https://ramly.io).

© Ramly LLC. Skill content is MIT-licensed. The Ramly service is subject to its terms.
