---
layout: default
title: Portfolio | Tutorials
---

[Home](../../index.md) /

### Tutorials

{% assign tutorial_pages = site.pages
  | where_exp: "p", "p.dir contains page.dir" %}

{% assign all_tags = "" | split: "" %}

{% for p in tutorial_pages %}
  {% unless p.name == "index.md" %}
    {% if p.tags %}
      {% assign all_tags = all_tags | concat: p.tags %}
    {% endif %}
  {% endunless %}
{% endfor %}

{% assign all_tags = all_tags | uniq | sort %}

<select id="tag-filter" class="dropdown">
  <option value="all">All</option>
    {% for tag in all_tags %}<option value="{{ tag }}">{{ tag }}</option>{% endfor %}
</select>


<table class="table-card">
  <tr>
    <td class="table-card-full" style="width:1000px">

      <div class="tutorial-scroll">
        {% for p in tutorial_pages %}
          {% unless p.name == "index.md" %}
            <div class="tutorial-entry"
                 data-tags="{% if p.tags %}{{ p.tags | join: ',' }}{% endif %}">
              <a href="{{ p.url | relative_url }}">
                {{ p.title
                  | default: p.name
                  | replace: ".md", ""
                  | replace: "Portfolio | ", ""
                  | replace: "Tutorial | ", "" }}
              </a>
            </div>
          {% endunless %}
        {% endfor %}

      </div>

    </td>
  </tr>
</table>

<script>
    document.addEventListener("DOMContentLoaded", () => {
    const filter = document.getElementById("tag-filter");

    filter.addEventListener("change", () => {
        const selected = filter.value;
        document.querySelectorAll(".tutorial-entry").forEach(entry => {
            if(selected === "all"){
                entry.style.display = "";
                return;
            }
            const tags = entry.dataset.tags
                .split(",")
                .map(t => t.trim());

            entry.style.display =
                tags.includes(selected) ? "" : "none";
        });
    });
});
</script>