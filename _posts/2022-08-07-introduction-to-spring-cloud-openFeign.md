---
layout: post
title: Spring Cloud OpenFeign 소개
tags: [spring, spring-cloud, feign]
comments: true
---


# 1. 개요

* [Spring Cloud OpenFeign](https://spring.io/projects/spring-cloud-openfeign)의 기본 내용을 학습한다.

# 2. Dependencies

* 먼저 스프링 부트 웹 프로젝트를 하나 생성하고, Dependencies 를 아래와 같이 구성하자.
  * 웹 프로젝트로 생성하는 이유는 컨트롤러를 구현해서 테스트를 편하게 하기 위함이다.
  * build.gradle
    ```java
    dependencies {
      implementation 'org.springframework.boot:spring-boot-starter-web'
      implementation 'org.springframework.cloud:spring-cloud-starter-openfeign' // openfeign 을 사용하기 위한 최소한의 Dependency

      testImplementation 'org.springframework.boot:spring-boot-starter-test'
    }
  ```

# 3. Feign Client

* [Spring Cloud OpenFeign](https://spring.io/projects/spring-cloud-openfeign) 이란?
  * **스프링 부트 Application 에서 사용하는 REST 클라이언트** 를 말한다.
  * 어노테이션을 선언해서 사용하며, 인터페이스 형태로 구현한다.
* Feign을 사용하면, 인터페이스 정의를 위한 코드 외에 부가적인 코드를 작성하지 않아도 된다.

## 3.1. @EnableFeignClients

* main 함수가 있는 클래스에 @EnableFeignClients 를 추가하자.
* 이 애너테이션을 사용해야 Feign 클라이언트 컴포넌트를 스캔할 수 있다.
  ```java
  @EnableFeignClients // 추가해주어야 함
  @SpringBootApplication
  public class FeignApplication {
  
    public static void main(String[] args) {
      SpringApplication.run(FeignApplication.class, args);
    }
  }
  ```
  
## 3.2. @FeignClient
* 3.1. 를 마친 후 아래와 같이 Feign 클라이언트를 추가하자.
  ```java
  @FeignClient(value = "jplaceholder", url = "https://jsonplaceholder.typicode.com")
  public interface JSONPlaceHolderClient {

    @GetMapping(value = "/posts")
    List<Post> getPosts();

    @GetMapping(value = "/posts/{postId}")
    Post getPostById(@PathVariable("postId") Long postId);
  }
  ```
* 여기에서는 [JSONPlaceholder API](https://jsonplaceholder.typicode.com/) 를 사용하는 Feign 클라이언트를 정의하였다.
* @FeignClient 의 **value** 인수는 필수값이며, 임의의 이름을 설정할 수 있다.
* **url** 에는 API 의 base URL을 지정한다.

## 3.3. DTO, 컨트롤러, 서비스 구현
* 해당 섹션은 FeignClient 를 사용하는 데 필수적인 부분은 아니고 간단한 요청/응답 테스트를 하기 위해 구현하는 부분이다.
  * Post (JSONPlaceholder API 에서 사용하는 DTO)
    ```java
    public class Post {
      Long userId;
      Long id;
      String title;
      String body;
      boolean completed;
        
      public Long getUserId() {
        return userId;
      }
        
      public Long getId() {
        return id;
      }
        
      public String getTitle() {
        return title;
      }
        
      public String getBody() {
        return body;
      }
        
      public boolean isCompleted() {
        return completed;
      }
    }
    ```
  * PostService
    ```java
    @Service
    public class PostService {
      private final JSONPlaceHolderClient jsonPlaceHolderClient;
  
      public PostService(final JSONPlaceHolderClient jsonPlaceHolderClient) {
        this.jsonPlaceHolderClient = jsonPlaceHolderClient;
      }
  
      public List<Post> getPosts() {
        return jsonPlaceHolderClient.getPosts();
      }
  
      public Post getPostById(Long postId) {
        return jsonPlaceHolderClient.getPostById(postId);
      }
    }
    ```
  * PostController
    ```java
    @RestController
    public class PostController {
      private final PostService postService;
      
      public PostController(PostService postService) {
      this.postService = postService;
      }
      
      @GetMapping(value = "/posts")
      public List<Post> getPosts() {
        return postService.getPosts();
      }
      
      @GetMapping(value = "/posts/{postId}")
      public Post getPosts(@PathVariable(value = "postId") Long postId) {
        return postService.getPostById(postId);
      }
    }
    ```

## 3.4. API 요청 테스트
* 스프링 부트 애플리케이션의 main 함수를 실행하자. 
* 애플리케이션이 실행되면, 3.3. 에서 구현한 PostController 의 두 가지 엔드포인트로 요청을 보내자.
  * (1) **http://localhost:8080/posts** 로 요청
  * 응답 데이터 (2022.08.16 기준)
    ```json
    [
      {
        "userId": 1,
        "id": 1,
        "title": "sunt aut facere repellat provident occaecati excepturi optio reprehenderit",
        "body": "quia et suscipit\nsuscipit recusandae consequuntur expedita et cum\nreprehenderit molestiae ut ut quas totam\nnostrum rerum est autem sunt rem eveniet architecto",
        "completed": false
      },
      {
        "userId": 1,
        "id": 2,
        "title": "qui est esse",
        "body": "est rerum tempore vitae\nsequi sint nihil reprehenderit dolor beatae ea dolores neque\nfugiat blanditiis voluptate porro vel nihil molestiae ut reiciendis\nqui aperiam non debitis possimus qui neque nisi nulla",
        "completed": false
      }, ... ,
      {
        "userId": 10,
        "id": 100,
        "title": "at nam consequatur ea labore ea harum",
        "body": "cupiditate quo est a modi nesciunt soluta\nipsa voluptas error itaque dicta in\nautem qui minus magnam et distinctio eum\naccusamus ratione error aut",
        "completed": false
      }
    ]
    ```
  * (2) **http://localhost:8080/posts/7** 로 요청
  * 응답 데이터 (2022.08.16 기준)
    ```json
    {
      "userId": 1,
      "id": 7,
      "title": "magnam facilis autem",
      "body": "dolore placeat quibusdam ea quo vitae\nmagni quis enim qui quis quo nemo aut saepe\nquidem repellat excepturi ut quia\nsunt ut sequi eos ea sed quas",
      "completed": false
    }
    ```
    

# 4. 출처

* [Introduction to Spring Cloud OpenFeign](https://www.baeldung.com/spring-cloud-openfeign)
