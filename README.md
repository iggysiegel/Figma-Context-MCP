# Figma for Government MCP

An MCP server that gives AI coding agents access to Figma design data, so they can implement a
design from the source file rather than from a screenshot.

A fork of [GLips/Figma-Context-MCP](https://github.com/GLips/Figma-Context-MCP) by Graham Lipsman.
Nearly all of this codebase — the design extraction and simplification — is his work; the fork
changes the API host and a few related strings.

## Why this fork exists

Figma does not offer its remote MCP server to Figma for Government, and its desktop MCP server
requires a desktop app distributed from `figma.com`, which is blocked on many government networks.
That leaves those networks unable to use either option.

This server needs neither. Figma for Government exposes the same REST API on a different hostname,
so the only host it has to reach is `api.figma-gov.com`.

## What it does

Given a file key, and optionally a node ID, it fetches the design and strips the response down to
what matters for writing code: layout, styling, text, and component metadata. Raw Figma responses
are large and mostly irrelevant to implementation, and passing all of it to a model wastes
context. It exposes two tools:

- `get_figma_data` — fetch and simplify a whole file or a single node.
- `download_figma_images` — download image fills and rendered nodes to a local directory.

## Setup

Requires Node 20.20.0 or newer and a Figma for Government account.

### 1. Create an access token

In your account: Settings, then Security, then Personal access tokens. Grant the
`file_content:read` scope; it covers everything this server does.

### 2. Build

```bash
npm install
npm run build
```

### 3. Configure your MCP client

Point it at the built entry point over stdio, using an absolute path:

```
node /absolute/path/to/dist/bin.js --stdio --image-dir=/path/to/your/project/src/assets/figma
```

Pass `FIGMA_API_KEY` through the client's environment block rather than as an argument, so the
token does not appear in process listings.

Set `--image-dir` explicitly. It defaults to the server's working directory, which for a stdio
server is wherever the client happened to launch it. Downloads outside that directory are refused,
so it is a containment boundary worth pointing somewhere deliberate.

## Usage

Paste a `figma-gov.com` link into your agent's chat and ask it to implement the design.

Design data can be large. Select a frame and copy the link to it rather than linking a whole file;
leave the output format at its default, `tree`, which is about half the size of `yaml`; and if a
frame is still too big, `depth` limits how far down the node tree the server walks.

## Differences from upstream

The API host `https://api.figma-gov.com` is hardcoded, so this build cannot reach commercial
`figma.com` files — use upstream for those. URL parsing accepts `figma-gov.com` hostnames.

Telemetry is opt-in rather than opt-out. Nothing is sent unless `FRAMELINK_TELEMETRY=on` is set,
and with it unset no analytics client is constructed at all.

## Troubleshooting

A 403 with `"Invalid token"` means the token is wrong or missing `file_content:read`; a 403
mentioning permissions means the token's account cannot see that file. The server prints the full
list of likely causes with every 403.

To separate a network problem from a client problem, fetch a file directly, bypassing MCP:

```bash
node dist/bin.js fetch "https://www.figma-gov.com/design/<fileKey>/Name?node-id=1-2" \
  --figma-api-key=<token>
```
