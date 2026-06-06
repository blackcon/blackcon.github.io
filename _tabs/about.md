---
title: About
icon: fas fa-user
order: 1
---

I'm **Jihwan Yoon** (*blackcon*), a solo offensive security researcher based in
South Korea 🇰🇷. I work across exploit development, reversing, and vulnerability
research — with a focus on **AI / LLM infrastructure, hypervisors, and the trust
boundaries software relies on**.

## Focus areas

- **AI / LLM security** — LLM proxies, agents, and the Model Context Protocol (MCP)
- **Virtualization** — Hyper-V and VMware internals, hypercall fuzzing
- **Exploitation & reversing** — memory-corruption primitives, firmware, hardware

## Selected research &amp; disclosures

- **LiteLLM — Jinja2 SSTI → unauthenticated RCE** *(2026)*
  A single unauthenticated request reached `uid=0` on the LiteLLM proxy. Silently
  patched in [PR #25445](https://github.com/BerriAI/litellm/pull/25445) days before
  a planned Pwn2Own Berlin 2026 entry.
  → [Write-up](/posts/litellm-jinja2-ssti/)

- **Claude Code — trust-prompt bypass ×3** *(2026)*
  Three independent paths to silent shell execution from a cloned repository.
  Reported via HackerOne; closed by Anthropic as **Informative / intended behavior**.
  → [Write-up](/posts/claude-code-trust-bypass-trilogy/)

- **MCP — Tool Poisoning &amp; Advanced Tool Poisoning** *(2025)*
  Instruction injection through MCP tool descriptions, and the harder variant that
  hides in tool *outputs* to defeat description-only defenses.
  → [TPA](/posts/MCP-tool-poison-attack/) · [ATPA](/posts/mcp-security-atpa/)

- **HVFUZZ — Hyper-V hypercall fuzzer** *(2022)*
  A fuzzer built on hAFL2 for hunting bugs in the Microsoft virtualization stack.
  → [Write-up](/posts/HVFUZZ/) · [GitHub](https://github.com/blackcon/HVFUZZ)

More projects and PoCs are on the [Projects](/projects/) page.

## Contact

- **Email** — [{{ site.data.authors.author.bio.email }}](mailto:{{ site.data.authors.author.bio.email }})
- **GitHub** — [github.com/blackcon](https://github.com/blackcon)
- **LinkedIn** — [in/{{ site.data.authors.author.sns.linkedin }}](https://www.linkedin.com/in/{{ site.data.authors.author.sns.linkedin }}/)
- **X / Twitter** — [@{{ site.data.authors.author.sns.twitter }}](https://twitter.com/{{ site.data.authors.author.sns.twitter }})
