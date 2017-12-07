---
layout: default
---

<body>
  <div class="index-wrapper">
    <div class="aside">
        <div class="info-card">
        <h1>My Activiti</h1>
        <a href="https://github.com/limeng32/mybatis.flying/" target="_blank"><img src="https://cdn2.iconfinder.com/data/icons/social-icons-33/128/Github-32.png" alt="" width="24"/></a>
        <a href="https://maven-badges.herokuapp.com/maven-central/com.github.limeng32/mybatis.flying" target="_blank"><img src="https://maven-badges.herokuapp.com/maven-central/com.github.limeng32/mybatis.flying/badge.svg" alt="" /></a>
      </div>
      <div id="particles-js">
      </div>
    </div>
    <div class="index-content">
      <ul class="artical-list">
        {% for post in site.categories.blog %}
        <li>
          <a href="{{ site.url }}{{ post.url }}" class="title">{{ post.title }}</a>
          <div class="title-desc">{{ post.description }}</div>
        </li>
        {% endfor %}
      </ul>
    </div>
  </div>
</body>
