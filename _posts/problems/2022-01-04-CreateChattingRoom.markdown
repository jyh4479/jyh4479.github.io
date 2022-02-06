---
layout: problems
title: 채팅방 생성 관리에 대한 고찰
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
    
    <br><br>
  
    1. 멤버 id 정보 받기
        + addChatRoom.java (Controller)
        
       ```java
        @PostMapping(value = "/chatroom")
        public ResponseEntity<?> addChatRoom(@RequestBody List<Member> memberList) {
        log.info("run addChatRoom in Controller");
            try {
                chattingRoomService.createChattingRoom(memberList);
                return new ResponseEntity<>(null, null, HttpStatus.OK);
            } catch (Exception e) {
                return new ResponseEntity<>(null, null, HttpStatus.INTERNAL_SERVER_ERROR);
            }
        }
        ```
    
    <br><br>
  
    2. 채팅방 생성
        + createChattingRoom.java (Service)
        
        ```java
        @Transactional(rollbackOn = {Exception.class})
        public void createChattingRoom(List<Member> memberList) {

        ChattingRoom chattingRoom = new ChattingRoom();
        chattingRoomRepository.save(chattingRoom);

        NewChattingRoomInfo newChattingRoomInfo = new NewChattingRoomInfo();
        newChattingRoomInfo.setId(chattingRoom.getId());
        newChattingRoomInfo.setMemberList(memberList);
        
        webClient.post()
                .uri("/chattingroom")
                .bodyValue(newChattingRoomInfo)
                .retrieve()
                .onStatus(HttpStatus::isError, clientResponse -> Mono.error(Exception::new))
                .bodyToMono(boolean.class)
                .block();
        }
        ```
    
    <br><br>
  
    3. JSON 형태로 멤버에게 추가된 채팅방 정보 저장
        + addChattingRoom.java (Service)
        
        ```java
        @Transactional(rollbackFor = Exception.class)
        public void addChattingRoom(NewChattingRoomInfo newChattingRoomInfo) throws ParseException {
            Long roomId = newChattingRoomInfo.getId();
            List<MemberId> memberList = newChattingRoomInfo.getMemberList();

            JSONParser jsonParser = new JSONParser();
            ObjectMapper mapper = new ObjectMapper();

            for (MemberId memberId : memberList) {
                Member member = memberRepository.getById(memberId.getId());
    
                JSONArray jsonArray;
                if (member.getChattingRoomList() == null) jsonArray = new JSONArray();
                else {
                    Object obj = jsonParser.parse(member.getChattingRoomList());
                    JSONObject jsonObject = (JSONObject) obj;
                    jsonArray = (JSONArray) jsonObject.get("dataList");
                }
    
                JSONObject chattingList = new JSONObject();
    
                JSONObject newRoomInfo = new JSONObject();
                newRoomInfo.put("id", roomId);
    
                jsonArray.add(newRoomInfo);
                chattingList.put("dataList", jsonArray);
    
                member.setChattingRoomList(chattingList.toJSONString());
                memberRepository.save(member);
            }
        }
        ```
    
    > 프로젝트를 진행하면서 @Transactional에 대한 동작이 이해가지 않는 부분이 있었다.   
    > 보통 마지막에 Commit 작업을 통해 쿼리를 날리는 방식으로 알고있었는데 채팅방을 생성하고나서 바로 채팅방 id에 접근할 수 있었다.  
    > Auto increment 설정된 id를 가지고 있는 인스턴스를 생성할때 쿼리를 날리는 시점이 다르다고 예상하고있지만 @Transactional 어노테이션에 대한 공부가 부족하다고 판단된다.  
    > 해당 어노테이션을 가지고 학습한 내용을 포스팅할 계획이다.

<br><br><br>

+ JSON이 다른 좋은방법은 무엇이 있을까?