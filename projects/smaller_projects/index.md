---
layout: default
title: Portfolio | Smaller Projects
---

[Home](../../index.md) /

### Smaller Projects
These projects represent small <1000 lines of code projects to demonstrate my ability to program using a given language.

<div class="navbutton-container">
{% assign pages_in_folder = site.pages | where_exp: "p", "p.dir == page.dir" %}
{% assign pages_in_folder = pages_in_folder | sort: "name" %}

{% for p in pages_in_folder %}
  {% unless p.name == "index.md" %}
    <a href="{{ p.url | relative_url }}" class="navbutton">
      {{ p.title | default: p.name 
        | replace: ".md", "" 
        | replace: "Portfolio | ", "" }}
    </a>
  {% endunless %}
{% endfor %}
</div>

{% assign subdirs = site.pages | where_exp: "p", "p.dir contains page.dir and p.dir != page.dir" | map: "dir" | uniq %}

{% for dir in subdirs %}
  {% assign folder_name = dir | split: "/" | last %}
  {% assign files = site.pages | where_exp: "p", "p.dir == dir" %}
  {% assign files = files | sort: "name" %}

  <table class="table-card">
    <tr>
      <td class="table-card-full" style="width:1000px">
        <strong>{{ folder_name | replace: "-", " " | capitalize }}</strong><br><br>
        <div class="navbutton-container">
        {% for f in files %}
          {% unless f.name == "index.md" %}
            <a href="{{ f.url | relative_url }}" class="navbutton">
              {{ f.title | default: f.name 
                | replace: ".md", "" 
                | replace: "Portfolio | ", "" }}
            </a>
          {% endunless %}
        {% endfor %}
        </div>
      </td>
    </tr>
  </table>

{% endfor %}