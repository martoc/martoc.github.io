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

## Introducing Reveille

[Reveille](https://github.com/martoc/reveille) is a gateway that sits between your AI clients and your MCP servers. The name comes from the bugle call that wakes soldiers in the morning, and that's exactly its job: **servers sleep until somebody needs them, then Reveille wakes them up.**

![Many AI sessions connect to one Reveille gateway, which shares a single copy of each server: the servers in use are awake and the rest sleep until called](https://martoc.github.io/blog/images/reveille-overview.svg)

Here's what it gives you.

### One shared copy of every server

However many sessions you open, each server runs at most once. Fifty sessions talking to the Helm documentation server share a single instance. Your laptop's memory no longer depends on how many terminals you have open.

### Servers sleep when nobody is using them

Servers start on the first real request and go back to sleep after a period of inactivity. A typical day might wake four or five servers, while the rest never start at all. When everything is asleep, the only thing running is the gateway itself.

### Connecting never wakes anyone up

When a client connects, it asks every server for its list of tools. With a naive approach, that alone would wake every server, every time you open a session. Reveille remembers each server's tools, so connecting and browsing what's available is instant and costs nothing. A server wakes only when the model actually wants to *use* it.

### One front door with a small menu

Every server's tools take up space in the model's context window. With twenty servers that adds up, and it leaves less room for your actual work. Reveille can present all your servers through a single entry point with just three tools: list the sources, list a source's tools, and call one. The model discovers what it needs as it goes, and your context window stays clean.

You can also connect to individual servers directly. You choose per client.

### Works with the servers you already have

Reveille doesn't ask anything of your MCP servers. Local programs, container images and remote HTTP servers all work unchanged, including third-party servers you didn't write. If it speaks MCP, Reveille can share it.

### From laptop to cluster, with the same behaviour

Reveille runs in two places, and the behaviour is identical:

- **On a laptop**, it reads a folder of simple configuration files and runs servers locally.
- **On Kubernetes**, each server is declared as a resource. Reveille scales its workload from zero to one when it's needed and back to zero when it isn't. It's packaged as a Helm chart and works on any conformant cluster, whether managed, self-hosted or local.

You can start on your laptop and move the same set of servers to a cluster later without changing how your clients connect.

### Secure by default

A gateway that fronts tools for an AI deserves careful handling:

- Every request needs a token, and the gateway refuses to listen on a network address without one.
- On a laptop, it only listens locally unless you tell it otherwise.
- Browser-based requests from unexpected origins are rejected.
- Credentials for servers such as Jira or GitHub come from the environment or Kubernetes secrets. They're never written into configuration, and request contents and secrets are never logged.

### Honest when things go wrong

Waking a server can fail: an image that can't be pulled, a missing secret, a process that crashes on start. Reveille doesn't leave the model waiting on a vague timeout. It reports *why* the server couldn't start, in words that a human (or a model) can act on. Status, metrics and health checks show what's awake, what's asleep and how long each wake took.

### Ready for the next version of MCP

The MCP specification keeps evolving. Reveille speaks both the established revision and the newest one to clients, and translates for servers that only know the older one. Your clients can move to the new protocol without waiting for every server to catch up.

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

When authoritative documentation is published this way, at a predictable address and in a format a model can read cheaply, much of what my documentation servers do becomes unnecessary. An agent could fetch the official source directly and trust it.

We're not there yet. Coverage is patchy, conventions are still settling, and a model fetching pages on its own still has to work out which sources are authoritative. Until AI-friendly documentation is the norm, the do-it-yourself approach fills the gap: index the official docs, serve them through MCP, and make them cheap to keep within reach. That's why these servers exist, and why Reveille exists to share them.

If the web catches up and makes them redundant, I'll happily retire them. That would be the best outcome.

## What's next

Reveille is complete on laptops and on Kubernetes, and it has been rehearsed with the full set of 22 servers. The next step is running it on a small home cluster and measuring it there. After that, I plan to open-source it on GitHub.

The [previous post](https://martoc.github.io/2026/02/13/mcp-servers-documentation/) ended with the idea that the future of AI-assisted development isn't AI that knows everything, but AI that knows where to look. Reveille is the practical follow-up. **The library stays open for every session, and the lights only come on in the rooms you're actually reading.**
