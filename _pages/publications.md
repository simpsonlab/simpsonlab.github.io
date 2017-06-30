---
title: "Simpson Lab - Publications"
layout: gridlay
excerpt: "Simpson Lab -- Publications."
sitemap: false
permalink: /publications/
---

# Publications

This list is also available on [Google Scholar](https://scholar.google.ca/citations?user=jcbxkKwAAAAJ)

## Preprints

{% for publi in site.data.publist %}

  {% if publi.preprint == 1 %}
  {{ publi.title }} <br />
  <em>{{ publi.authors }} </em><br />
  <a href="{{ publi.link.url }}">{{ publi.link.display }}</a> ({{publi.year}})
  {% endif %}

{% endfor %}

## Peer Reviewed

{% for publi in site.data.publist %}

  {% if publi.preprint == 0 %}
  {{ publi.title }} <br />
  <em>{{ publi.authors }} </em><br />
  <a href="{{ publi.link.url }}">{{ publi.link.display }}</a> ({{publi.year}})
  {% endif %}

{% endfor %}

