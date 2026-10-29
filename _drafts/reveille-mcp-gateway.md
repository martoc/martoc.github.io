---
title: Reveille - MCP Servers That Sleep Until You Need Them
subtitle: Sharing documentation MCP servers across every AI session, without melting your laptop
layout: post
author: martoc
image: https://martoc.github.io/blog/images/Claude_AI_symbol.svg
---

In [my previous post](https://martoc.github.io/2026/02/13/mcp-servers-documentation/) I described a set of open-source MCP servers that let an AI assistant search and read the official documentation for tools like Helm, Spark, ClickHouse and Karpenter. The idea was simple: teach the AI to read the manual before it answers.

The servers worked. Running them was the hard part. This post covers what went wrong once I used them every day, what I learnt from it, and the gateway I built as a result: **Reveille**.

## Why documentation servers matter

Before the plumbing, it's worth explaining why this is worth the effort at all.

You might expect a capable model with web search to be enough. In my experience it isn't, for two reasons.

**Knowledge goes stale quickly.** APIs, configuration options and best practices change with every release. A model's training data is a snapshot, and some of it is already out of date by the time the model ships. Ask about a flag that was renamed six months ago and you'll often get the old name, delivered with complete confidence.

**The internet isn't a reliable source.** Search results mix official documentation with outdated blog posts, half-finished forum answers and plain guesswork. A model has no reliable way to tell an authoritative page from a confident stranger. Here's a thought experiment. Invent a technology that doesn't exist, publish a few convincing pages about it, then ask an AI assistant to help you use it. More often than not it will follow the fake instructions enthusiastically, when it should stop you and say *"this doesn't exist"*.

The fix is the same advice we give junior engineers: **read the book, not the crazy people on the internet**. A documentation MCP server gives the model one curated, versioned, authoritative source to consult, so its answers are grounded in what the project actually says. It hallucinates less, guesses less, and stops being a magnificent liar about niche configuration options.

This only works if the documentation servers are *always available* to every session you open. That's where the trouble started.

## Lesson one: one container per session doesn't scale

My first setup used the Docker Desktop MCP Toolkit with Claude. It's a lovely experience for a single conversation: pick a server from the catalogue and it just works.

The catch is how it works. **Every session starts its own copy of every server.** One Claude session with a dozen documentation servers means a dozen containers. Open a few terminals, a couple of agents running in parallel, an IDE integration, and the numbers multiply fast.

Each documentation server uses roughly 70 MiB of memory once it's warm. At 50 sessions and 15 servers, that's 750 containers and around **51 GiB of memory**, nearly all of it doing nothing. I didn't need a spreadsheet to discover this: my laptop crashed first.

## Lesson two: running everything all the time is wasteful

The obvious next step was to flip the model around. I ran every server once, permanently, behind a fixed facade, and pointed every session at those shared instances.

That solved the crash. Every session shared the same servers, so the number of sessions no longer mattered. But it brought a quieter problem: **everything ran all the time**, and most of it was rarely used.

On any given day I might need the Kubernetes and Helm documentation, and not Spark, Airflow or Cloud Custodian. They ran anyway. With 22 servers, the always-on stack used about **2.5 GiB of memory at idle**. That's a steep standing cost for documentation I wasn't reading.

The two lessons together pointed to what I actually wanted:

- **Shared**, like the facade, so the number of sessions doesn't matter.
- **On demand**, like a single session, so idle servers cost nothing.

## What Reveille is

[Reveille](https://github.com/martoc/reveille) is a gateway for MCP servers. It sits between your AI clients (Claude Code, Claude Desktop, IDE assistants, agents) and the MCP servers they use, and it makes those servers cheap to share.

The name comes from the bugle call that wakes soldiers in the morning, and that's exactly its job: **servers sleep until somebody needs them, then Reveille wakes them up.**

![Many AI sessions connect to one Reveille gateway, which shares a single copy of each server: the servers in use are awake and the rest sleep until called](https://martoc.github.io/blog/images/reveille-overview.svg)

Three ideas do most of the work:

- **One shared copy of every server.** However many sessions you open, each server runs at most once. Fifty sessions talking to the Helm documentation server share a single instance, so your memory no longer depends on how many terminals you have open.
- **Servers sleep when nobody is using them.** A server starts on the first real call and stops after a period of inactivity. A typical day might wake four or five servers, while the rest never start at all. When everything is asleep, only the gateway runs.
- **Connecting never wakes anyone up.** When a client connects, it asks every server for its list of tools. Done naively, that alone would wake every server every time you open a session. Reveille remembers each server's tools after it first meets them, so from then on connecting is instant and free, and a server wakes only when the model actually wants to *use* it.

## How it works

You don't need to know any of this to use Reveille, but it helps explain why it behaves the way it does.

### Every server gets its own address

Each server is published at its own path on the gateway, such as `/kubernetes-documentation` or `/helm-documentation`, using MCP's standard Streamable HTTP transport. To a client, it looks exactly like connecting to the server directly. It doesn't know or care that the server is shared, asleep or running somewhere else.

### Many conversations share one connection

Behind each path, Reveille keeps a single connection to the real server and multiplexes every client's requests over it. It tags each request so the answer goes back to the session that asked, and leaves everything else in the message untouched. Progress updates and cancellations are routed the same way, so long-running calls behave as they would with a private server.

### A catalogue of every server's tools

The first time a server starts, Reveille records what it offers (its tools, prompts and resources) and keeps that catalogue on disk, or in the cluster on Kubernetes. From then on, clients connect, list and browse from the catalogue, even after the gateway restarts. When a server announces that its tools have changed, or its configuration changes, the catalogue is refreshed.

### Waking up, exactly once

When a call arrives for a sleeping server, Reveille starts it, and only once, even if twenty sessions ask at the same moment. While the server starts, the client's request is held open with regular keep-alive messages and a "starting server" progress note, so nothing times out. As soon as the server is ready, the real answer follows on the same connection. Typically this takes a couple of seconds.

### Going back to sleep

Each request counts as activity. Once a server has gone a set time without any (fifteen minutes by default, configurable per server), Reveille stops it gracefully. A server is never stopped while a call is still in flight.

### Looking after fragile servers

Some servers can only handle one call at a time. For those, Reveille queues the extra requests and keeps them alive while they wait, so the server isn't overwhelmed. If a server crashes, the calls in flight fail with the reason, including the exit code and the last thing the server said. Repeated failures back off gradually instead of restarting in a tight loop.

### One core, two places to run

The core of Reveille doesn't know where it's running. It depends on two things: somewhere to read the list of servers from, and a way to start and stop them. Swapping those two pieces is what lets the same gateway run in very different places:

![The same Reveille core runs as a single binary on a laptop or server, with servers as Docker containers or local processes, and on Kubernetes with servers as Deployments scaled between zero and one](https://martoc.github.io/blog/images/reveille-runtimes.svg)

- **On a laptop or a server**, Reveille is a single binary, and servers are defined in a folder of small configuration files. A server that names a container image runs as a Docker container, which Reveille creates, watches and removes itself by talking to Docker directly. Anything else runs as a local program: a Python tool through `uvx`, a Node.js package through `npx`, or whatever you like. Edit a file and Reveille picks up the change without a restart.
- **On Kubernetes**, each server is declared as a custom resource. Reveille turns it into a Deployment kept at zero replicas, scales it to one when a call arrives and back to zero when it goes idle. A tiny adapter inside each server's pod turns a standard input/output server into an HTTP one, so any existing server image works unchanged. Each server's state (asleep, waking, ready or failed) shows up in the cluster, next to everything else you run.

## Everything it does

| Capability | What it means for you |
|------------|-----------------------|
| **Shared servers** | Each server runs at most once, whatever the number of sessions. |
| **Sleep and wake on demand** | Servers start on the first real call and stop after a configurable idle time. |
| **Tool catalogue** | Once a server's tools are known, connecting and listing them never starts it, even after a restart. |
| **One address per server** | Each server keeps its own address and its own tools, so clients connect to it as if it were running just for them. |
| **Optional front door** | For large collections, one extra address with three tools (list the servers, list a server's tools, call a tool) reaches every server, so the model's context window stays small. |
| **Any MCP server** | Local programs, container images and remote HTTP servers work unchanged, including third-party servers. |
| **Keeps clients waiting politely** | Requests are held with keep-alives and progress while a server wakes, so clients don't time out. |
| **Queueing** | Servers that handle one call at a time get an orderly queue instead of being overwhelmed. |
| **Clear failures** | A server that can't start says why, whether it's a missing image, a missing secret, a crash or a timeout, in words a person or a model can act on. |
| **Crash recovery** | Failed servers back off gradually and recover on their own. |
| **Both MCP revisions** | Clients can speak the established revision or the newest stateless one. Reveille translates for servers that only know the older one. |
| **Safe defaults** | Local-only listening unless you choose otherwise, browser origin checks, secrets kept out of configuration, nothing sensitive in the logs, and an optional access token for when you share the gateway beyond your own machine. |
| **Live configuration** | Add, change or remove servers without restarting the gateway. A broken change is reported and the last good configuration keeps working. |
| **Observability** | Prometheus metrics, health checks, and a live view of each server's state, wake time and activity. Servers can also be woken, put to sleep or refreshed by hand. |
| **Runs anywhere** | As a single binary on Linux or macOS, or on any conformant Kubernetes cluster (managed, self-hosted or local), on Intel/AMD and ARM machines. |

## Getting started

There are two ways to run Reveille. Pick the one that matches what you already have.

### The binary: the quickest start

If you have Docker, you can run Reveille. Download the binary for your machine from the latest release (Linux or macOS, Intel or ARM), and start it with the example servers:

```sh
git clone https://github.com/martoc/reveille && cd reveille
os=$(uname -s | tr '[:upper:]' '[:lower:]')
arch=$(uname -m | sed 's/x86_64/amd64/; s/aarch64/arm64/')
curl -fsSL -o reveille "https://github.com/martoc/reveille/releases/latest/download/reveille-$os-$arch"
chmod +x reveille
./reveille serve --config-dir examples/servers
```

That starts the gateway with the public documentation servers configured, including the ones from my previous post, all asleep. Each one starts as a container on its first call and is removed when idle. Adding a server is a short file in the `examples/servers` folder, and the same file works unchanged on Kubernetes. Servers that aren't containers, such as `uvx` or `npx` tools, run as local programs.

### On Kubernetes, with Helm or kustomize

For a team, a home lab or anyone who already runs a cluster, Reveille installs with a Helm chart or a plain kustomize base. Servers are declared as Kubernetes resources, sleep at zero replicas, and are shared by every client that can reach the gateway:

```sh
helm install reveille oci://ghcr.io/martoc/charts/reveille \
  --namespace reveille-system --create-namespace
```

```yaml
apiVersion: reveille.martoc.io/v1alpha1
kind: MCPServer
metadata:
  name: helm-documentation
spec:
  path: /helm-documentation
  image: martoc/mcp-helm-documentation:latest
  command: ["/app/.venv/bin/mcp-helm-docs"]
  idleTimeout: 15m
```

Credentials for registries and servers come from whatever the cluster already provides: node identities, pull secrets and Kubernetes Secrets. Reveille doesn't handle any cloud's credentials itself.

### Connecting your AI client

Whichever way you run it, you connect Claude Code to each server at its own address, exactly as you would connect to the server itself. In the example servers every server's address is its name, so a short loop adds every documentation server:

```sh
for f in examples/servers/*-documentation.yaml; do
  name=$(basename "$f" .yaml)
  claude mcp add --scope user --transport http "$name" "http://127.0.0.1:8808/$name"
done
```

If you collect dozens of servers, you can instead add a single entry for the front door, `http://127.0.0.1:8808/mcp`, and let Claude discover the servers as it needs them.

Claude Desktop starts its servers as local programs, so it reaches each address through [`mcp-remote`](https://github.com/geelen/mcp-remote), a small bridge, with one entry per server. Desktop keeps a connection open to each one, and Reveille answers it without waking the server, so a sleeping server still shows as available instead of looking down.

### One gateway for all your machines

I run a single Reveille on a small always-on machine at home and use it from my laptop and everything else. Listen on an address your other machines can reach, and point the clients at it. A [Tailscale](https://tailscale.com/) address keeps the gateway off the local network and the internet:

```sh
# On the always-on machine (its address, from: tailscale ip -4):
./reveille serve --config-dir examples/servers --listen 100.101.102.103:8808

# On each client machine, one entry per documentation server:
for name in \
  a2a-documentation airflow-documentation aws-documentation \
  azure-documentation clickhouse-documentation cloudcustodian-documentation \
  go-documentation helm-documentation karpenter-documentation \
  keda-documentation kubernetes-documentation prometheus-documentation \
  spark-documentation strimzi-documentation; do
  claude mcp add --scope user --transport http "$name" "http://100.101.102.103:8808/$name"
done
```

Every machine then shares the same sleeping servers and the same tool catalogue, and only the always-on machine pays for the servers that are awake. Authentication is off by default. If more people than you can reach that address, start the gateway with an access token and have each client send it.

## The results

I measured the full set of 22 servers on a single machine, comparing the always-on facade with Reveille:

| Setup | Memory |
|-------|--------|
| One container per session and server (estimate, 50 sessions × 15 servers) | ~51 GiB |
| Every server always running (the fixed facade) | 2,553 MiB |
| Reveille, every server awake | 1,685 MiB |
| Reveille, every server asleep (the normal idle state) | **18 MiB** |

Reveille is cheaper even with everything awake, because it replaces heavyweight wrappers with a small adapter. The real win, though, is the last line: **when you're not using your documentation, it costs almost nothing.**

To check the sharing holds up under load, I ran 50 simultaneous sessions for ten minutes against every server, starting with all of them asleep:

- **16,128 operations, zero failures**, with no restarts and no out-of-memory kills.
- Calls to an awake server answered in **2 to 14 ms** typically, and never more than about 50 ms at the 99th percentile.
- The gateway itself stayed at around **50 MiB** throughout.

## The trade-offs

There are trade-offs, and they're worth stating plainly.

- **The first call is slower.** Waking a server takes a few seconds, typically two to three and up to about ten for the largest. After that it's as fast as ever. For documentation lookups this is a good trade, and you can tune how long each server stays awake.
- **Servers are shared.** That's the point, but it means a server that needs credentials uses one identity for everyone connected to it. That's fine for a personal setup or a team with shared access. It's not a way to give each user their own permissions.
- **It's deliberately focused.** Reveille shares servers efficiently and leaves features like per-session server state and subscriptions to the servers themselves.

## A bridge, not a destination

All of this, the documentation servers and the gateway that shares them, solves a problem I hope won't last.

Today's web was built for human eyes. A typical documentation page is mostly HTML: navigation bars, cookie banners, scripts, styling and layout markup. The useful content is a small fraction of it. When a model reads a page like that, most of its tokens go on noise, and the structure that makes the page easy for a person to scan, such as headings, code blocks and tables, often gets lost on the way.

That's starting to change. More and more projects publish documentation in forms built for machines as well as people:

- **Markdown versions of pages**, served next to the HTML, so a model gets the content without the chrome.
- **`llms.txt`**, a [proposed convention](https://llmstxt.org/) for a curated, Markdown index of a site's most important documentation, placed where an AI agent knows to look.
- **Token-efficient formats** such as [TOON](https://github.com/toon-format/toon) (Token-Oriented Object Notation), which carry structured data in fewer tokens than JSON.
- **Official, hosted MCP servers** run by the vendors themselves. AWS offers the [AWS Knowledge MCP server](https://awslabs.github.io/mcp/servers/aws-knowledge-mcp-server) for searching and reading its documentation, and it's available as a ready-made connector in Claude. Microsoft does the same for its documentation with the [Microsoft Learn MCP server](https://github.com/MicrosoftDocs/mcp). There's nothing to install or run: you connect, and the publisher keeps the content current.

When authoritative documentation is published this way, at a predictable address and in a format a model can read cheaply, much of what my documentation servers do becomes unnecessary. An agent could fetch the official source directly and trust it. A vendor-hosted MCP server goes one step further, because the people who write the docs also serve them.

We're not there yet. Coverage is patchy, and most projects, especially open-source tools like Helm, Spark or Karpenter, don't host anything like this. Conventions are still settling, and a model fetching pages on its own still has to work out which sources are authoritative. Until AI-friendly documentation is the norm, the do-it-yourself approach fills the gap: index the official docs, serve them through MCP, and make them cheap to keep within reach. That's why these servers exist, and why Reveille exists to share them.

If the web catches up and makes them redundant, I'll happily retire them. That would be the best outcome.

## What's next

Reveille runs as a single binary and on Kubernetes, and it has been rehearsed with the full set of 22 servers. The next step is running it on a small home cluster and measuring it there. After that, I plan to open-source it on GitHub, and I hope a single download makes it easy for anyone to try.

The [previous post](https://martoc.github.io/2026/02/13/mcp-servers-documentation/) ended with the idea that the future of AI-assisted development isn't AI that knows everything, but AI that knows where to look. Reveille is the practical follow-up. **The library stays open for every session, and the lights only come on in the rooms you're actually reading.**
