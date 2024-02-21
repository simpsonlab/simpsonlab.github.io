---
title: "Simpson Lab - Team"
layout: gridlay
excerpt: "Simpson Lab: Team members"
sitemap: false
permalink: /team/
---

# Group Members

{% assign number_printed = 0 %}
{% for member in site.data.team_members %}

{% assign even_odd = number_printed | modulo: 2 %}

{% if even_odd == 0 %}
<div class="row">
{% endif %}

<div class="col-sm-6 clearfix">
  <img src="{{ site.url }}{{ site.baseurl }}/assets/teampic/{{ member.photo }}" class="img-responsive" width="25%" style="float: left" />
  <h4>{{ member.name }}</h4> 
  {% if member.github != 0 %}
  <a href="{{ member.github }}/"> <i class="fa fa-github" style="color:black; font-size:24px;"></i></a>
  {% endif %}
  {% if member.twitter != 0 %}
  <a href="{{ member.twitter }}/"> <i class="fa fa-twitter" style="color:#0084b4; font-size:24px;"></i></a>
  {% endif %}
  <br>
  <i>{{ member.info }}<br></i>
  <ul style="overflow: hidden">
  
  {% if member.number_educ == 1 %}
  <li> {{ member.education1 }} </li>
  {% endif %}
  
  {% if member.number_educ == 2 %}
  <li> {{ member.education1 }} </li>
  <li> {{ member.education2 }} </li>
  {% endif %}
  
  {% if member.number_educ == 3 %}
  <li> {{ member.education1 }} </li>
  <li> {{ member.education2 }} </li>
  <li> {{ member.education3 }} </li>
  {% endif %}
  
  {% if member.number_educ == 4 %}
  <li> {{ member.education1 }} </li>
  <li> {{ member.education2 }} </li>
  <li> {{ member.education3 }} </li>
  <li> {{ member.education4 }} </li>
  {% endif %}
  
  </ul>
</div>

{% assign number_printed = number_printed | plus: 1 %}

{% if even_odd == 1 %}
</div>
{% endif %}

{% endfor %}

{% assign even_odd = number_printed | modulo: 2 %}
{% if even_odd == 1 %}
</div>
{% endif %}


## Alumni
<table align="center" class="table table-condensed">
<tr>
    <th>Staff and Postdocs</th>
</tr>
  <tr><td>Sabiq Chaudhary, 2020-2023</td></tr>
  <tr><td>Richard de Borja, 2020-2023</td></tr>
  <tr><td>Michael Molnar, 2018-2022</td></tr>
  <tr><td>Paul Tang, 2016-2020</td></tr>
  <tr><td>Ina Anreiter, 2018-2019</td></tr>
  <tr><td>Hamza Khan, 2017-2019</td></tr>
  <tr><td>Jonathan Dursi, 2015-2016</td></tr>
  <tr><td>Matei David, 2013-2017</td></tr>
  <tr><td>Marina Barsky, 2013-2015</td></tr>
<tr>
    <th>Graduate Students</th>
</tr>
  <tr><td>Alister-D'Costa, PhD, 2017-2024</td></tr>
  <tr><td>Jonathan Broadbent, MSc, 2021-2023</td></tr>
  <tr><td>Heather Gibling, PhD, 2016-2022</td></tr>
  <tr><td>Joanna Pineda, MSc, 2017-2020</td></tr>
<tr>
    <th>Undergraduate Students</th>
</tr>
  <tr><td>Jun Gao, 2023</td></tr>
  <tr><td>Yasamin Nouri Jelyani, 2023</td></tr>
  <tr><td>Sean Gong, 2022</td></tr>
  <tr><td>Bianca Pokhrel, 2019</td></tr>
  <tr><td>Tina Lee, 2018</td></tr>
  <tr><td>Yin Yin, 2017</td></tr>
  <tr><td>Audrina Zhou, 2017</td></tr>
  <tr><td>Tom Piao, 2016</td></tr>

</table>

## Visitors

<table align="center" class="table table-condensed">
<tr>
    <td>Kiran Garimella (University of Oxford) 2018</td>
</tr>
</table>

<br />
