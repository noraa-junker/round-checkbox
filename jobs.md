---
layout: page
title: My (previous) jobs and work experience
permalink: /about/jobs/
regenerate: true
---

<a href="/about/">&lt;&lt;&lt; Back to about</a>

Here is a list of all my job history. As I am still a student, all of these are part-time jobs.
{% assign work = site.work | reverse %}
{% for job in work %}

<h3>{{ job.title }} </h3>
<p><b>At: {{ job.place }}</b></p>
<p><b>Time: {{ job.time }}</b></p>
<p>{{ job.description }}</p>

{% endfor %}