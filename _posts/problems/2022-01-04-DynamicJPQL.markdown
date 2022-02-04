---
layout: problems 
title: 동적으로 JPA 사용하기 
date: 2022-01-04 19:20:23 +0900 
category: problems
---
# 1. 문제 생각하기
> 업무 관련 프로젝트를 진행하면서 JPA 기반으로 작성된 게시판에서 옵션을 추가하여 검색을 해야 하는 문제가 있었다.  
> 기존 JPQL을 사용하였을 때는 옵션을 넣는 경우마다 분기하여 처리된 코드를 보면서 
> 동적으로 JPQL을 사용하는 방법은 없을까 의문을 가지면서 게시물을 작성하게 되었다.

+ JPA 기반 옵션을 통한 검색 기능을 구현하고자 할 때 동적으로 검색하기 위한 기술이 무엇이 있을까?
+ QueryDSL을 사용하는 방법과 간단한 예시

# 2. 해결방법을 위한 과정

> 사내에서 진행하는 Spring 스터디 시간에 위에서 언급했던 것 처럼 옵션 개수에 따라 JPQL 쿼리 개수가 2^n 만큼 생성되어야하는 문제에 대해 언급하고 다른 방법이 없는지 질문하게 되었다.  
> 다른 개발자 분이 'QueryDSL'을 사용하면 해결할 수 있을 것이라고 알려주셨고,
> '동적 JPA' 라는 키워드로 구글링을 해보니 QueryDSL에 대한 정보들이 많이 나와있었다.
> 관련 정보들을 나름대로 정리해보고 적용해보았다.

# 3. 해결방법

> QueryDSL을 적용해보고 옵션 검색 기능을 구현해보자

+ QueryDSL 환경 설정
  > spring boot 2.6.2, QueryDSL 5.0.0 기준으로 구글에 나와있는 많은 환경 설정 방법으로 build.gradle을 작성하면 오류가 발생하는 문제가 있었다.  
  > 조금 더 찾아본 결과 해당 위에 적어놓은 버전을 사용하는 경우 아래와 같이 build.gradle을 작성해야한다. (버전에 따라 환경설정은 바뀔수 있다.)

  + build.gradle
    <pre>
      <code>
      buildscript {
        ext {
          queryDslVersion = "5.0.0"
        }
      }
  
      plugins {
        id "com.ewerk.gradle.plugins.querydsl" version "1.0.10"
      }
  
      dependencies {
        implementation "com.querydsl:querydsl-jpa:${queryDslVersion}"
        implementation "com.querydsl:querydsl-apt:${queryDslVersion}"
      }
  
      def querydslDir = "$buildDir/generated/querydsl"
  
      querydsl {
        jpa = true
        querydslSourcesDir = querydslDir
      }
  
      sourceSets {
        main.java.srcDir querydslDir
      }
  
      compileQuerydsl {
        options.annotationProcessorPath = configurations.querydsl
      }
  
      configurations {
        compileOnly {
          extendsFrom annotationProcessor
        }
        querydsl.extendsFrom compileClasspath
      }
      </code>
    </pre>

  > JPAQueryFactory를 Bean에 등록하기 위한 설정이다.  
  > EntityManager를 아래와 같이 인자로 전달하여 Bean에 등록하면 된다.
  + QueryDslConfig.java
    
    ```java
      @Configuration
      public class QueryDslConfig {
      @PersistenceContext
      private EntityManager entityManager;
  
      @Bean
      public JPAQueryFactory jpaQueryFactory() {
          return new JPAQueryFactory(entityManager);
      }
    }
    ```
  
+ Entity
  > QueryDSL 환경을 갖추고 빌드를 하면 build > generated > querydsl 경로로 @Entity를 타겟하여 Q Entity를 생성한다.  
  > Q Entiy는 Bean에 등록한 JPAQueryFactory에 사용된다. 아래는 타겟이되는 Entity다.
  
  + Content.java (Entity)
    ```java
      @Data
      @Table(name = "content")
      @Entity
      public class Content {
      @Id
      @GeneratedValue(strategy = GenerationType.IDENTITY)
      private Long id;
      
          @Column
          private String writer;
      
          @Column
          private String title;
      
          @Column
          private String detail;
      
          @Column
          private String date;
      
      
          @OneToMany(mappedBy = "content")
          @JsonManagedReference
          private List<Comment> commentList = new ArrayList<Comment>();
      
          @Column
          private String comment;
      
          @Builder
          public Content(String writer, String title, String detail, String date, String comment) {
              this.writer = writer;
              this.title = title;
              this.detail = detail;
              this.date = date;
              this.comment = comment;
          }
      
          public Content() {
      
          }
      }
      ```
+ Repository
  > QueryDSL을 사용하는데 있어 가장 중요한 부분이라고 생각된다.  
  > 소스코드를 통해 기존 JPA의 Repository와 QueryDSL을 사용하기 위해 생성한 RepositoryCustom이 어떻게 연관이 있는지,
  > 실제 구현이 어떻게 이루어지는지 유심히 보면 좋을것같다.
  + ContentRepository.java (interface)
    ```java
    @Repository
    public interface ContentRepository extends JpaRepository<Content, Long>, ContentRepositoryCustom {
    }
    ```
  + ContentRepositoryCustom.java (interface)
    ```java
    public interface ContentRepositoryCustom {
    List<Content> findByOption(Pageable paging, SearchOption searchOption);
      
    Integer findByOptionSize(SearchOption searchOption);
    }
    ```
  + ContentRepositoryImpl.java
    ```java
    @Log
    @AllArgsConstructor
    public class ContentRepositoryImpl implements ContentRepositoryCustom {
    private final JPAQueryFactory jpaQueryFactory;
    
        private BooleanExpression eqId(Long id) {
            if (id == null) return null;
            else return QContent.content.id.eq(id);
        }
    
        private BooleanExpression containWriter(String writer) {
            if (writer == null) return null;
            else return QContent.content.writer.contains(writer);
        }
    
        private BooleanExpression containTitle(String title) {
            if (title == null) return null;
            else return QContent.content.title.contains(title);
        }
    
        private BooleanExpression eqDate(String date) {
            if (date == null) return null;
            else return QContent.content.date.eq(date);
        }
    
        public List<Content> findByOption(Pageable paging, SearchOption searchOption) {
            log.info("run ContentRepositoryImpl findByOption");
            List<Content> result = jpaQueryFactory.selectFrom(QContent.content)
                    .where(eqId(searchOption.getId()),
                            containTitle(searchOption.getTitle()),
                            eqDate(searchOption.getDate()),
                            containWriter(searchOption.getWriter()))
                    .offset(paging.getPageNumber() * paging.getPageSize())
                    .limit(paging.getPageSize())
                    .fetch();
            return result;
        }
    
        public Integer findByOptionSize(SearchOption searchOption) {
            List<Content> result = jpaQueryFactory.selectFrom(QContent.content)
                    .where(eqId(searchOption.getId()),
                            containTitle(searchOption.getTitle()),
                            eqDate(searchOption.getDate()),
                            containWriter(searchOption.getWriter()))
                    .fetch();
            return result.size();
        }
    }
    ```
    > 기존에 옵션개수에 따라 모든 Query를 구현하는 것이 아닌 BooleanExpression을 사용함으로써
    > 옵션의 null 체크를 하고 null인 경우는 옵션의 모든 정보를 검색하도록한다.  
    > (추가적으로 BooleanExpression이 아닌 그냥 null을 where절에 입력하는경우 오류가 발생)