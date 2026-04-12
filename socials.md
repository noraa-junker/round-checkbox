---
layout: page
title: My social media
permalink: /about/socials/
regenerate: true
---

<a href="/about/">&lt;&lt;&lt; Back to about</a>

Here is a list of all my social media channels and what purpose they serve for myself:

{% for social in site.socials %}

<h3>{{ social.name }} (<a href="{{social.link}}">{{social.username}}</a>)</h3>
<p>{{ social.description }}</p>
<p><a href="{{social.link}}">{{social.link | replace: "https://" ""}}</a></p>


{% endfor %}