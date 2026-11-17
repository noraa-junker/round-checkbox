---
layout: page
title: My (previous) jobs and work experience
regenerate: true
---

{% include aboutBackLink.md %}

Here is a list of all my job history. As I am still a student, all of these are part-time jobs.
{% assign work = site.work | reverse %}
{% for job in work %}

<h3>{{ job.title }} </h3>
<p><b>At: {{ job.place }}</b></p>
<p><b>Time: {{ job.time }}</b></p>
<p>{{ job.description }}</p>

{% endfor %}