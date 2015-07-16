---
layout: page
title: Sick of Acronyms
tagline: Scattered Thoughts
---

#Michael Ramirez

Michael is founder of the Greenville, SC based startup, Apothesource. He is currently focused on developing [PillFill](http://www.pillfill.com), a set of Android and iOS applications that aim to simplify health data management for patients.

<p align="center">
<img src="/img/michael-grady.png">
</p>


From 2012 through 2013, Michael was Chief Engineer for the **VA/DoD iEHR project** for *Space and Naval Warfare Systems Command* (SPAWAR), an effort intended to create one of the largest EHR systems in the world. He previously was Chief Engineer for the **VA Post 9/11 GI Bill Benefits System** (Chapter 33 LTS) from 2009 to 2012, where he led efforts that delivered the first modern, SOA-based benefits management system for the Department of Veterans Affairs (VA).


#Latest Posts:

<ul class="posts">
  {% for post in site.posts limit:3 %}
    <li><span>{{ post.date | date_to_string }}</span> &raquo; <a href="{{ BASE_PATH }}{{ post.url }}">{{ post.title }}</a></li>
  {% endfor %}
</ul>




