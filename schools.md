---
layout: page
title: My schools and education history
permalink: /about/schools/
regenerate: true
---

<a href="/about/">&lt;&lt;&lt; Back to about</a>

Here is a list of all my education history.
{% assign schools = site.schools | reverse %}
{% for school in schools %}

<h3>{{ school.name }} </h3>
<p><b>Category: {{ school.category }}</b></p>
<p><b>Time: {{ school.time }}</b></p>
<p>{{ school.Description }}</p>
<p><a href="{{school.website}}">{{school.website | replace: "https://" ""}}</a></p>


{% endfor %}