---
title: MCP Servers - Teaching AI to Read the Manual (So You Don't Have To)
subtitle: Building open-source documentation MCP servers for cloud-native tools
layout: post
author: martoc
image: https://martoc.github.io/blog/images/Claude_AI_symbol.svg
---

We've all been there. It's 2am, you're knee-deep in a ClickHouse query that refuses to cooperate, the Helm chart values file looks like it was written in ancient Sumerian, and your AI assistant confidently suggests a Karpenter configuration option that hasn't existed since version 0.27. The AI is brilliant, creative, and occasionally a magnificent liar when it comes to niche technical documentation.

What if your AI assistant could actually _read the docs_ before answering? Not the docs from two years ago baked into its training data, but the real, current, authoritative documentation? That's precisely what the Model Context Protocol (MCP) enables, and it's why I've built six open-source MCP servers for the tools I use every day.

## What is MCP?

The [Model Context Protocol](https://modelcontextprotocol.io/) is an open standard introduced by [Anthropic in November 2024](https://www.anthropic.com/news/model-context-protocol) that standardises how AI systems integrate with external data sources and tools. Think of it as a USB-C port for AI: one universal connector instead of a drawer full of proprietary cables.

At its core, MCP uses a client-server architecture. AI applications (clients) connect to data sources and tools (servers) through a standard protocol built on JSON-RPC. When an agent needs data, context, or to perform an action, it sends a standardised request to an MCP server. The server retrieves the information or executes the action, then sends back a standardised response. Simple, elegant, and remarkably powerful.

MCP defines three core primitives:

| Primitive | Description | Controlled By |
|-----------|-------------|---------------|
| **Tools** | Executable functions that AI applications can invoke | Model |
| **Resources** | Data sources that provide contextual information | Application |
| **Prompts** | Reusable templates for structuring interactions | User |

The protocol supports multiple transport mechanisms. **STDIO** runs the server as a local subprocess, ideal for command-line tools. **Streamable HTTP** enables remote deployment with support for multiple client connections. The older **SSE** transport has been [deprecated in favour of Streamable HTTP](https://modelcontextprotocol.io/specification/2025-03-26/basic/transports).

## Why MCP Matters

The adoption numbers speak for themselves. MCP has grown from an internal Anthropic experiment to an [industry standard](https://www.pento.ai/blog/a-year-of-mcp-2025-review) with over 1,000 community-built servers. [OpenAI adopted MCP in March 2025](https://developers.openai.com/apps-sdk/concepts/mcp-server/), [Google Cloud](https://cloud.google.com/discover/what-is-model-context-protocol) followed suit, and Microsoft Azure [incorporated MCP into the Azure AI Agent Service](https://www.thoughtworks.com/en-us/insights/blog/generative-ai/model-context-protocol-mcp-impact-2025). Organisations implementing MCP report 40-60% faster agent deployment times.

But beyond the industry momentum, there are practical reasons why MCP is a game-changer for developers:

- **No more hallucinated documentation**: Your AI assistant queries real, indexed documentation instead of relying on training data that may be outdated or incomplete
- **Build once, use everywhere**: A single MCP server works across Claude Desktop, Claude Code, Cursor, VS Code, and any MCP-compatible client
- **Context efficiency**: Tools load on demand rather than consuming precious context window space upfront
- **Composability**: Chain multiple MCP servers together, giving your AI access to documentation, databases, APIs, and custom tools simultaneously

## Best Practices for Building MCP Servers

Having built six documentation MCP servers, I've learned a few things about what makes a good one. Here are the practices that have worked well.

### Keep Tools Focused and Minimal

Each MCP server should expose a small number of well-defined tools. All my servers expose exactly two tools:

- `search_documentation` - Full-text search with optional filtering
- `read_documentation` - Retrieve complete document content

This follows the principle of least surprise. The AI model doesn't need to reason about which of fifteen tools to call; it searches, finds what it needs, and reads the full document.

### Pre-Index Documentation at Build Time

Rather than cloning repositories and indexing at runtime, build Docker images with pre-indexed documentation. This means instant startup, consistent performance, and no network dependency at query time. Every one of my servers uses a multi-stage Dockerfile that indexes documentation during the build phase.

### Use Efficient Search with BM25 Ranking

SQLite FTS5 with BM25 ranking is a remarkably capable search engine for this use case. With weighted fields (title: 5.0, description: 2.0, content: 1.0) and Porter stemming, search results are relevant and fast (typically under 100ms). The key is proper query sanitisation to prevent FTS5 syntax errors from user input.

### Sparse Git Checkout for Efficiency

Most documentation repositories are enormous. Apache Spark's repository is over 1GB, but the docs directory is only around 10MB. Using sparse checkout with shallow cloning (`--depth 1`) dramatically reduces both clone time and disk usage.

### Security Considerations

MCP servers can execute commands and perform API calls, which means security cannot be an afterthought. The [MCP specification includes security best practices](https://modelcontextprotocol.io/specification/draft/basic/security_best_practices) that should be followed:

- Use parameterised SQL queries to prevent injection
- Validate and cap all user inputs (search limits between 1-50)
- Use subprocess list arguments rather than shell strings
- Follow least-privilege principles

A notable [incident in September 2025](https://www.practical-devsecops.com/mcp-security-vulnerabilities/) involved an unofficial MCP server that silently added BCC fields to emails, leaking content to an attacker. Always review MCP servers you install, especially those from untrusted sources.

## My Open-Source MCP Documentation Servers

I've built a suite of six MCP servers, all following the same architecture and quality standards. Each one indexes official documentation from the upstream project and makes it searchable through Claude.

### Common Architecture

All servers share a consistent design:

```text
MCP Client (Claude Desktop / Claude Code / Cursor)
    |
    | STDIO Transport (JSON-RPC)
    |
FastMCP Server (2 tools)
    |
DocumentDatabase (SQLite FTS5, BM25 ranking)
    |
Indexer (sparse git checkout)
    |
Parser (RST / Markdown / MDX)
```

Each server is built with Python 3.12+, uses [FastMCP](https://github.com/jlowin/fastmcp) as the framework, and includes comprehensive test suites with pytest. They all ship as Docker images on Docker Hub for easy deployment.

### The Servers

#### mcp-airflow-documentation

Indexes **Apache Airflow** core documentation (RST format) and the Python Client documentation (Markdown format) into a single unified database.

- **Sources**: Airflow Core (~300 docs) + Python Client (~100 docs)
- **Parsers**: RST (docutils with visitor pattern) + Markdown (frontmatter)
- **Unique feature**: Source-aware filtering - search across both sources or filter by `airflow-core` or `airflow-python-client`
- **Repository**: [github.com/martoc/mcp-airflow-documentation](https://github.com/martoc/mcp-airflow-documentation)
- **Docker**: `martoc/mcp-airflow-documentation:latest`

#### mcp-clickhouse-documentation

Indexes **ClickHouse** documentation including the knowledge base articles. Handles Docusaurus MDX files with JSX components.

- **Sources**: ClickHouse docs + knowledge base (~850 docs)
- **Parser**: MDX with JSX component cleaning (regex-based, no Node.js dependency)
- **Unique feature**: Multi-directory indexing covering both `docs/` and `knowledgebase/`
- **Repository**: [github.com/martoc/mcp-clickhouse-documentation](https://github.com/martoc/mcp-clickhouse-documentation)
- **Docker**: `martoc/mcp-clickhouse-documentation:latest`

#### mcp-cloudcustodian-documentation

Indexes **Cloud Custodian** documentation with section-based filtering by cloud provider.

- **Sources**: Cloud Custodian docs (~195 docs)
- **Parser**: RST with docutils visitor pattern
- **Unique feature**: Section filtering by cloud provider (AWS, Azure, GCP, Kubernetes, OCI)
- **Repository**: [github.com/martoc/mcp-cloudcustodian-documentation](https://github.com/martoc/mcp-cloudcustodian-documentation)
- **Docker**: `martoc/mcp-cloudcustodian-documentation:latest`

#### mcp-helm-documentation

Indexes **Helm** documentation from the helm-www repository.

- **Sources**: Helm docs (~130 docs)
- **Parser**: Markdown with YAML frontmatter and MDX cleaning
- **Unique feature**: Rich section structure including chart template guide and best practices
- **Repository**: [github.com/martoc/mcp-helm-documentation](https://github.com/martoc/mcp-helm-documentation)
- **Docker**: `martoc/mcp-helm-documentation:latest`

#### mcp-karpenter-documentation

Indexes **AWS Karpenter** documentation from the karpenter-provider-aws repository.

- **Sources**: Karpenter website docs (~181 docs)
- **Parser**: Markdown with Hugo shortcode cleaning
- **Unique feature**: Hugo-specific content handling and karpenter.sh URL generation
- **Repository**: [github.com/martoc/mcp-karpenter-documentation](https://github.com/martoc/mcp-karpenter-documentation)
- **Docker**: `martoc/mcp-karpenter-documentation:latest`

#### mcp-spark-documentation

Indexes **Apache Spark** documentation with section-based filtering.

- **Sources**: Spark docs (sql-ref, streaming, mllib, graphx, and more)
- **Parser**: Markdown with Jekyll liquid tag cleaning
- **Unique feature**: Extensive section coverage including SQL reference, ML library, and graph processing
- **Repository**: [github.com/martoc/mcp-spark-documentation](https://github.com/martoc/mcp-spark-documentation)
- **Docker**: `martoc/mcp-spark-documentation:latest`

## Getting Started

Setting up any of these servers takes about 30 seconds. Add the following to your Claude Desktop configuration (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "airflow-docs": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "martoc/mcp-airflow-documentation:latest"]
    },
    "clickhouse-docs": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "martoc/mcp-clickhouse-documentation:latest"]
    },
    "cloudcustodian-docs": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "martoc/mcp-cloudcustodian-documentation:latest"]
    },
    "helm-docs": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "martoc/mcp-helm-documentation:latest"]
    },
    "karpenter-docs": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "martoc/mcp-karpenter-documentation:latest"]
    },
    "spark-docs": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "martoc/mcp-spark-documentation:latest"]
    }
  }
}
```

For Claude Code, add a `.mcp.json` file to your project root:

```json
{
  "mcpServers": {
    "helm-docs": {
      "command": "docker",
      "args": ["run", "-i", "--rm", "martoc/mcp-helm-documentation:latest"]
    }
  }
}
```

Once configured, you can simply ask Claude questions and it will automatically search the relevant documentation. For example:

> "How do I configure a Karpenter NodePool with spot instances?"

Claude will call `search_documentation` on the Karpenter MCP server, find the relevant pages, call `read_documentation` to get the full content, and provide an answer grounded in the actual current documentation.

## Contributing

All six servers are open-source under the MIT licence. Contributions are welcome, whether that's reporting issues, suggesting new features, or submitting pull requests. The common architecture means that once you understand one server, contributing to any of them is straightforward.

If you'd like to build your own documentation MCP server, the pattern is well-established:

1. Create a parser for your documentation format (RST, Markdown, MDX, or HTML)
2. Build an indexer that clones and processes the upstream repository
3. Store documents in SQLite FTS5 with BM25 ranking
4. Expose `search_documentation` and `read_documentation` tools via FastMCP
5. Package it in a Docker image with pre-indexed documentation

The future of AI-assisted development isn't about AI that knows everything. It's about AI that knows where to look and how to read. MCP makes that future practical today.
