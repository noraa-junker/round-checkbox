---
layout: page
title: My awards and certifications
permalink: /about/certs/
regenerate: true
---

<a href="/about/">&lt;&lt;&lt; Back to about</a>

Here is a list of all awards I received and certifications I gained through the years.
{% assign certs = site.certs | reverse %}
{% for cert in certs %}

<h3>{{ cert.title }} </h3>
<p><b>{{ cert.time }}</b></p>
<p><a href="{{cert.link}}">{{cert.link | replace: "https://" ""}}</a></p>
<p>{{ cert.description }}</p>

{% endfor %}