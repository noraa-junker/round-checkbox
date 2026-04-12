---
layout: page
title: The programming languages I can program with
permalink: /about/languages/
regenerate: true
---

<a href="/about/">&lt;&lt;&lt; Back to about</a>

Here is a list of all the programming and markup languages I can program with. 

{% assign markuplanguages = site.langs | where: 'isMarkup', true %}
{% assign graphiclanguages = site.langs | where: 'isGraphical', true %}
{% assign programminglanguages = site.langs | where: 'isGraphical', false | where: 'isMarkup', false %}

<h2>Markup languages</h2>
{% for lang in markuplanguages %}

<h3>{{ lang.name }} </h3>
<p>{{ lang.description }}</p>

{% endfor %}

<h2>Graphical languages</h2>
{% for lang in graphiclanguages %}
<h3>{{ lang.name }} </h3>
<p>{{ lang.description }}</p>

{% endfor %}

<h2>Programming languages</h2>
{% for lang in programminglanguages %}

<h3>{{ lang.name }} </h3>
<p>{{ lang.description }}</p>

{% endfor %}
