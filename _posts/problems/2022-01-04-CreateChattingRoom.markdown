---
layout: problems
title: DB에 JSON 데이터 저장과 Entity 인스턴스 생성 원리에 대한 고찰
date: 2022-01-04 19:20:23 +0900
category: problems
---
# 1. 문제 생각하기
> 개인 프로젝트로 채팅 서비스를 구현하는중 채팅방(Entity)을 Micro Service 구조에서 어떻게 생성하고 관리해야 효율적인지  
> 추가적으로 JPA를 사용하였을때 Entity 생성 원리에 대한 고민을 하게되었다.

+ 사용자와 채팅방에 대한 DB에서의 정보관리
+ 채팅방의 ID를 Auto Increment로 설정할 때 JPA 동작 원리

<br><br><br>

# 2. 해결방법을 위한 과정
> JPA에서 테이블을 매핑하기 위한 관련 Annotation(@OneToMany, @ManyToMany 등)에 대한 존재는 알고 있었지만 왠지 모르게
> 해당 문제를 해결하기에 앞서 JSON자체를 DB에 저장하고 관리하면 어떨까라는 생각을 먼저하게 되어 그렇게 구현하게 되었다.
> (나중에 JSON자체를 저장하는것에 대한 장단점을 찾아보게되었고 현재 프로젝트 로직도 바꾸고 포스팅해볼 계획이다.)

<br><br><br>

# 3. 해결방법
> 위에서 언급했던 것처럼 가장 처음 구현한 방법은 JSON을 DB에 저장하는 방식으로 채팅방 생성 로직을 구현하였다.

+ JSON을 이용한 채팅방 정보 
    > JSON을 이용하는 경우 각각 Member는 채팅방에 대한 정보를 아래와 같이 String으로 저장한다.
    + Member.java (Entity)
    ```java
    @Entity
    @Table(name = "member")
    @Data
    @NoArgsConstructor(access = AccessLevel.PROTECTED)
    public class Member {
  
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private String id;
  
    // ...other info
  
    @Column
    @JsonRawValue
    private String chattingRoomList;  
    }
    ```
    
    <br><br>
    > 다음으로 채팅방에 대한 정보를 담을 Entity이다. (실질적으로 별다른 기능이 없어서 id만 가지고있는 상태다.)
    + Member.java (Entity)
    ```java
    @Entity
    @Table(name = "chatting_room")
    @Data
    @NoArgsConstructor(access = AccessLevel.PUBLIC)
    public class ChattingRoom {
  
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    }
    ```
  
    <br><br>
    + logic
      > 1. 생성되는 채팅방에 속하게될 멤버 id 정보를 받는다.
      > 2. 채팅방을 생성한다.
      > 3. 생성된 채팅방의 id를 해당하는 멤버에게 JSON 형태로 저장한다.

    1.  