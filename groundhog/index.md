---
layout: page
title: Groundhog
---

Local AI personal data pipeline. Consolidates health (Garmin), stocks, and reminders into a local DuckDB database — queryable via natural language using a local LLM.

**Stack:** Python · DuckDB · Ollama · Qwen3-VL · qwen2.5-coder:32b

---

## Journal

<ul>
{% for post in site.posts %}
  {% if post.categories contains 'groundhog' %}
  <li>
    <a href="{{ post.url }}">{{ post.date | date: "%Y-%m-%d" }} — {{ post.title }}</a>
  </li>
  {% endif %}
{% endfor %}
</ul>
