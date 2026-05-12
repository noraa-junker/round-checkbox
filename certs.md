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
<div style="border-radius: 25px; border: 2px solid black; padding: 10px;
            display: flex; align-items: flex-start; gap: 15px;">
<img src="/images/{{cert.image}}" width="50px" style="flex-shrink: 0;margin-top:8px">
<div>
<h3>{{ cert.title }} </h3>
<p><b>{{ cert.time }}</b></p>
{%if cert.link != null %}<p><a href="{{cert.link}}">Link to credential</a></p>{%endif%}
<p>{{ cert.description }}</p>
</div>
</div>  
{% endfor %}