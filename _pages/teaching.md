---
title: "Simpson Lab - Teaching"
layout: gridlay
excerpt: "Simpson Lab -- Teaching."
sitemap: false
permalink: /teaching/
---

# Teaching

### University of Toronto Courses

{% assign uoft_courses = site.data.courselist | where: "type", "course" %}
{% for course in uoft_courses %}

  <a href="{{ course.link.url }}">{{ course.link.display }} ({{course.term}}) - {{ course.title }} </a>

{% endfor %}

### Canadian Bioinformatics Workshops

{% assign cbw_courses = site.data.courselist | where: "type", "cbw" %}
{% for course in cbw_courses %}
  <a href="{{ course.link.url }}"> {{ course.title }} ({{course.term}})</a>
{% endfor %}

<br>
