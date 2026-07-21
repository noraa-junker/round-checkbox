---
layout: page
regenerate: true
title: Articles mentioning me
---

<a href="/about/">&lt;&lt;&lt; Back to about</a>

Here is a list of articles, videos, etc. about me or my work:
{% assign mentionings = site.mentionings | sort: 'date' | reverse %}
{%for m in mentionings %}

<h3>{{m.name}}</h3>
<p><b>From: {{m.date | date: "%B %e %Y"}}</b></p>
<p>{{m.description}}</p>
<p><a href="{{m.link}}">{{m.link | replace: "https://" ""}}</a></p>
{%endfor%}