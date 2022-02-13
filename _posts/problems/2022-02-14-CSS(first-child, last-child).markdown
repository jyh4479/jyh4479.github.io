---
layout: problems title: CSS(first-child, last-child)
date: 2022-02-14 19:20:23 +0900 category: problems
---

# 1. 문제 생각하기

> 개인 이력서를 위해 react로 Card component를 구현하는 중 css에 border-radius를 적용하게되었다.
> Card component내부에 title, content, image, footer 등 여러 component로 구성되어 있었고,
> 생각 없이 title 윗 부분에 radius, footer 아랫 부분에 radius를 적용하게 되었다.  
> 하지만 title을 사용하지 않는 경우와 footer를 사용하지 않는 경우도 생각하면 radius가 적용되지 않은 component(content 등)가
> radius가 적용된 부분을 벗어나는 문제가 발생하여 해결하기 위한 방법과 과정을 포스팅하기 위해 글을 작성한다.

<br><br><br>

# 2. 해결방법을 위한 과정

> 결론부터 말하면 css를 제대로 공부하지 않아서 맞게된 문제다.
> BootStrap에서 제공하는 Card Component가 위에서 언급한 문제를 잘 해결해놓아서 개발자 도구로 적용된 css를 살펴보았다.
> first-child, last-child를 적용하여 다이나믹하게 radius를 적용하였고 바로 프로젝트 소스 코드에 옮겨서 결과를 확인하였다.

<br><br><br>

# 3. 해결방법

> css의 first-child와 last-child를 적용하고 관련 기능에 대해 학습해보자
