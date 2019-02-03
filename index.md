# TODO
1. 상단에 View on github 제거하기 -> theme 파일을 다 리포에 받은 뒤, 커스텀
2. 잘 안쓰지만 쓸법한 코드들 모으는 포스트 만들기
3. 프로젝트 로그들 백업
4. json 관리 프로그램 제작


<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>
