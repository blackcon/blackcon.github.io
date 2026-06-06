---
title: About
icon: fas fa-user
order: 1
---

I'm **Jihwan Yoon** (*blackcon*), a solo offensive security researcher based in
South Korea 🇰🇷. I work across exploit development, reversing, and vulnerability
research — with a focus on **AI / LLM infrastructure, hypervisors, and the trust
boundaries software relies on**.

**Background:** [BoB](https://www.kitribob.kr/) 3rd (Best of the Best) · Naver Cloud /
NBP — see the full [career timeline](/career/).

## Focus areas

- **AI / LLM security** — LLM proxies, agents, and the Model Context Protocol (MCP)
- **Virtualization** — Hyper-V and VMware internals, hypercall fuzzing
- **Exploitation & reversing** — memory-corruption primitives, firmware, hardware

## Selected research &amp; disclosures

{% assign disclosures = site.posts | where_exp: "item", "item.tags contains 'disclosure'" %}
<ul class="bc-pub-list">
{% for post in disclosures %}
  <li class="bc-pub">
    <div class="bc-pub-meta">
      <span class="bc-pub-cat">{{ post.categories | join: " · " }}</span>
      <span class="bc-pub-date">{{ post.date | date: "%b %Y" }}</span>
    </div>
    <a class="bc-pub-title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
    {% if post.description %}
    <p class="bc-pub-desc">{{ post.description | strip_html | strip_newlines }}</p>
    {% endif %}
  </li>
{% endfor %}
</ul>

This list is generated automatically from posts tagged `disclosure`. More
projects and PoCs are on the [Projects](/projects/) page.

## Contact

- **Email** — [{{ site.data.authors.author.bio.email }}](mailto:{{ site.data.authors.author.bio.email }})
- **GitHub** — [github.com/blackcon](https://github.com/blackcon)
- **LinkedIn** — [in/{{ site.data.authors.author.sns.linkedin }}](https://www.linkedin.com/in/{{ site.data.authors.author.sns.linkedin }}/)
- **X / Twitter** — [@{{ site.data.authors.author.sns.twitter }}](https://twitter.com/{{ site.data.authors.author.sns.twitter }})
