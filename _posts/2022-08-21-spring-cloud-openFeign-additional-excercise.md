---
layout: post
title: Spring Cloud Feign Client 추가 실습
subtitle: Retryable, Recover 기능 및 WireMock 테스트 실습
tags: [spring, spring-cloud, feign]
comments: true
---

# 1. 개요

* Feign 클라이언트의 기본기는 [여기](https://bky373.github.io/2022-08-07-introduction-to-spring-cloud-openFeign/)에서 살펴봤다.
* 이번 문서에서는 기존 실습에 더해 **Retry 와 Recover 기능을 사용하는 방법** 에 대해 알아본다.
* 또한 **WireMock 을 사용하여 이들 기능을 모두 테스트하는 법** 에 대해 알아본다.
 
# 2. 프로젝트 구조

* 실습 파일을 만들 때 경로가 많이 헷갈리므로 프로젝트 구조를 미리 공유한다.
  ```markdown
    .
    ├─ main/java
    │        └─com.example.feign
    │              ├─ FeignApplication.java
    │              ├─ client
    │              │    ├─ BaseFeignClientPackage.java
    │              │    ├─ JsonFeignClient.java
    │              │    ├─ JsonFeignClientService.java
    │              │    └─ JsonFeignPost.java
    │              └─ config
    │                   ├─ FeignClientConfiguration.java
    │                   └─ FeignClientRetryLogger.java
    └─ test/java
            └─com.example.feign
                   └── client
                        └── JsonFeignClientTest.java
       
  ```

# 3. Dependency

* 모든 작업 전에 openfeign dependency 를 추가하자.
    * **build.gradle**
      ```
      dependencies {
      
        ...
      
        implementation("org.springframework.cloud:spring-cloud-starter-openfeign")
      
        ...
    
      }
      ```

# 4. Configuration

* Feign Client 를 사용하려면 @EnableFeignClients 를 등록해야 한다.
* Feign Client 전용 basePackage 와 Configuration 을 만들고, 이 안에 **@EnableFeignClients** 를 등록하자.
* 각 클래스의 위치는 처음 소개한 **[프로젝트 구조]** 를 참고한다.
    * **BaseFeignClientPackage**
      ```java
      public interface BaseFeignClientPackage {
      }
      ```
    * **FeignClientConfiguration**
      ```java
      @Configuration
      @ComponentScan(basePackageClasses = {BaseFeignClientPackage.class})
      @EnableFeignClients(basePackageClasses = {BaseFeignClientPackage.class})
      public class FeignClientConfiguration {
      }
      ```

# 5. Feign Client

* 이전에도 해봤지만 Feign Client 는 인터페이스 정의 와 어노테이션 설정만으로 간단히 구현이 가능하다.
* **JsonFeignClient** 를 정의하면서, Feign Client 통신에서 사용할 데이터 **JsonFeignPost** 를 정의하고, 이들을 사용할 **JsonFeignClientService** 를 정의하자.
    * **JsonFeignClient**
        ```java
        @FeignClient(value = "json-api", url = "${external.json-api.url}")
        public interface JsonFeignClient {
         
          @GetMapping(value = "/posts")
          List<JsonFeignPost> getPosts();
          
          @GetMapping(value = "/posts/{postId}")
          JsonFeignPost getPostById(@PathVariable("postId") Long postId);
        }
        ```
    * **JsonFeignPost**
        ```java
        public class JsonFeignPost {
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
    * **JsonFeignClientService**
      ```java
      @Component
      public class JsonFeignClientService {
        private final JsonFeignClient jsonFeignClient;
      
        public JsonFeignClientService(final JsonFeignClient jsonFeignClient) {
          this.jsonFeignClient = jsonFeignClient;
        }
        
        public List<JsonFeignPost> getPosts() {
          return jsonFeignClient.getPosts();
        }
        
        public JsonFeignPost getPostById(Long postId) {
          return jsonFeignClient.getPostById(postId);
        }
      }
      ```

# 6. Retry Configuration

* Feign 라이브러리에서 제공하는 Retry Bean 을 사용할 수도 있지만, 이는 글로벌 설정이라는 단점이 있다.
* 여기에서는 특정 상황에서만 Retry 를 시도할 수 있도록 스프링에서 제공하는 **@Retryable** 을 사용할 것이다.
* 이를 위해 스프링에서 제공하는 retry dependency 를 추가하고, 기능을 활성화하기 위해 **@EnableRetry** 를 등록하자.
    * **build.gradle**
       ```
       implementation("org.springframework.retry:spring-retry")
       ```
    * **FeignClientConfiguration**
        ```java
        @Configuration
        @EnableRetry(proxyTargetClass = true) // 새로 추가함
        @ComponentScan(basePackageClasses = {BaseFeignClientPackage.class})
        @EnableFeignClients(basePackageClasses = {BaseFeignClientPackage.class})
        public class FeignClientConfiguration {
        }
        ```
* 다음으로 Retry 기능을 사용할 메서드에 **@Retryable** 을 등록하자.
    * **JsonFeignClientService**
      ```java
      @Component
      public class JsonFeignClientService {
        private final JsonFeignClient jsonFeignClient;
        
        public JsonFeignClientService(final JsonFeignClient jsonFeignClient) {
          this.jsonFeignClient = jsonFeignClient;
        }
          
        @Retryable(
          value = FeignException.class, // FeignExcpetion 이 발생했을 때만 Retry 기능이 활성화된다.
          maxAttemptsExpression = "${external.json-api.retry.maxAttempts:3}",  // 총 3번 Retry 한다. 
          backoff = @Backoff(delayExpression = "${external.json-api.retry.maxDelay:1000}") // 매번 1초씩 sleep 후 Retry 한다. 
        )
        public List<JsonFeignPost> getPosts() {
          return jsonFeignClient.getPosts();
        }
          
        public JsonFeignPost getPostById(Long postId) {
          return jsonFeignClient.getPostById(postId);
        }
      }
      ```
* Retry 기능이 어떻게 동작하는지 확인하면 좋으니 Logger 도 정의한다.
    * **FeignClientRetryLogger**
      ```java
      @Component
      public class FeignClientRetryLogger extends RetryListenerSupport {
        @Override
        public <T, E extends Throwable> void onError(final RetryContext context,
                                                     final RetryCallback<T, E> callback,
                                                     final Throwable throwable) {
          System.out.printf( // 실무에서는 로그로 대체하자.
            "[RETRY_CONTEXT]-%s, [RETRY_COUNT]-%s, [EXCEPTION]-%s%n",
            context.getAttribute(RetryContext.NAME),
            context.getRetryCount(),
            throwable.getMessage()
          );
        }
      }
      ```

# 7. Test

* Retry 기능이 설정한 대로 잘 동작하는지 테스트할 때 Mock Server 를 사용하면 편리하다.
    * Mock Server 를 사용하면 200 응답이든, 502 응답이든 원하는 응답을 가정할 수 있기 때문이다.
* Mock Server 를 사용하려면 아래 test dependency 를 추가해야 한다.
    * (주의) 스프링 클라우드 버전에 따라 추가해야 할 dependency 가 다르다.
    * **build.gradle**
      ```
      // 스프링 클라우드 버전이 Hoxton 인 경우
      testImplementation("org.springframework.cloud:spring-cloud-starter-contract-stub-runner")
      
      // 최신 버전을 사용하는 경우 (확인한 건 2021.0.3 버전이었음)
      testImplementation("org.springframework.cloud:spring-cloud-contract-wiremock")
      ```
* 아래는 테스트 코드 이다.
    * **JsonFeignClientTest**
      ```java
      @AutoConfigureWireMock(port = 0) // Mock Server 자동 구성 설정
      @SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
      @TestPropertySource(properties = {
        "external.json-api.url=http://localhost:${wiremock.server.port}"
      })
      class JsonFeignClientTest {
        @Autowired
        private JsonFeignClientService jsonFeignClientService;
        
        @Test
        public void throwsFeignExcpetion() {
          // given
          stubFor(
            get("/posts")
              .willReturn(status(502)) // FeignException [502 BadGateway]
          );
        
          // when
          Throwable throwable = catchThrowable(jsonFeignClientService::getPosts);
        
          // then
          verify(
              3,
              getRequestedFor(urlEqualTo("/posts"))
          );
          assertThat(throwable).isInstanceOf(FeignException.class);
          assertThat(throwable.getMessage()).contains("502 Bad Gateway");
        }
      }
      ```
* 테스트를 실행하면 Mock Server 의 랜덤 포트로 요청이 나가고, retry 가 총 3번 수행된 것을 알 수 있다.
    * **Request received**
      ```
      2022-08-21 19:18:47.808  INFO 76687 --- [tp1678413715-50] WireMock                                 : Request received:
      127.0.0.1 - GET /posts
      
      Accept: [*/*]
      User-Agent: [Java/11.0.11]
      Connection: [keep-alive]
      Host: [localhost:10024]
      
      
      
      Matched response definition:
      {
      "status" : 502
      }
      
      Response:
      HTTP/1.1 502
      Matched-Stub-Id: [7855e4d8-6660-46d2-b259-3d400ed0f4b7]
      ```
    * **Retry 로그**
      ```
      [RETRY_METHOD]-public java.util.List<com.example.feign.client.JsonFeignPost> com.example.feign.client.JsonFeignClientService.getPosts(), [RETRY_COUNT]-1, [EXCEPTION]-[502 Bad Gateway] during [GET] to [http://localhost:10024/posts] [JsonFeignClient#getPosts()]: []
      ...
      [RETRY_METHOD]-public java.util.List<com.example.feign.client.JsonFeignPost> com.example.feign.client.JsonFeignClientService.getPosts(), [RETRY_COUNT]-2, [EXCEPTION]-[502 Bad Gateway] during [GET] to [http://localhost:10024/posts] [JsonFeignClient#getPosts()]: []
      ...
      [RETRY_METHOD]-public java.util.List<com.example.feign.client.JsonFeignPost> com.example.feign.client.JsonFeignClientService.getPosts(), [RETRY_COUNT]-3, [EXCEPTION]-[502 Bad Gateway] during [GET] to [http://localhost:10024/posts] [JsonFeignClient#getPosts()]: []
      ```
        * [RETRY_COUNT] 가 1씩 증가하며 3번 기록되었다.

# 8. Recover

* 모든 Retry 가 실패했을 때 특정 동작을 수행하고 싶다면 **@Recover** 를 사용할 수 있다.
    * 사용법은 간단하나, @Retryable 를 사용하는 메서드의 시그니처를 지켜야 한다는 점에 주의해야 한다.
    * 정확한 내용은 [이곳](https://docs.spring.io/spring-retry/docs/api/current/org/springframework/retry/annotation/Recover.html)을 참고하자.
* Recover 동작을 테스트하기 위해 일부 코드를 추가하였다.
    * **JsonFeignClientService**
      ```java
      @Component
      public class JsonFeignClientService {
        private final JsonFeignClient jsonFeignClient;
        
        ...
        
        // 새로운 테스트를 위해 getPostById 메서드에도 @Retryable 추가 
        @Retryable(
          value = FeignException.class, // FeignException 이 발생했을 때만 Retry 기능이 활성화된다.
          maxAttemptsExpression = "${external.json-api.retry.maxAttempts:3}",  // 총 3번 Retry 한다. 
          backoff = @Backoff(delayExpression = "${external.json-api.retry.maxDelay:1000}") // 매번 1초씩 sleep 후 Retry 한다. 
        )  
        public JsonFeignPost getPostById(Long postId) {
          return jsonFeignClient.getPostById(postId);
        }
      
        // Recover 수행 메서드 추가
        // 메서드 이름과 FeignException 인자를 제거하면 바로 위 getPostById 의 시그니처와 동일한 시그니처를 갖는다.   
        @Recover
        public JsonFeignPost afterAllRetries(FeignException exception, Long postId) {
          System.out.printf("##### Recover method called. exception = %s, postId = %d%n", exception, postId);
          return null;
        }
      }
      ```
    * **JsonFeignClientTest**
      ```java
      class JsonFeignClientTest {
        @Autowired
        private JsonFeignClientService jsonFeignClientService;
      
        @Test
        public void throwsFeignExcpetion() {
          // given
          stubFor(
            get("/posts/1")
              .willReturn(status(502)) // FeignException [502 BadGateway]
          );
        
          // when
          JsonFeignPost result = jsonFeignClientService.getPostById(1L);
        
          // then
          verify(
            3,
            getRequestedFor(urlEqualTo("/posts/1"))
          );
          assertThat(result).isEqualTo(null);
          }
        }
      ```
* 테스트 실행 로그를 보면, **모든 Retry 요청이 시도된 후 Recover 메서드가 수행** 되었다.
  ```
  ##### Recover method called. exception = feign.FeignException$BadGateway: [502 Bad Gateway] during [GET] to [http://localhost:11236/posts/1] [JsonFeignClient#getPostById(Long)]: [], postId = 1
  ```
* 또한 **메서드의 반환 값은 Recover 메서드의 반환 값** 을 취하고 있다.
  ```java
  assertThat(result).isEqualTo(null);
  ```

# 9. 참고 자료

* [Spring Cloud Feign Testing - Feign Client를 테스트해보자](https://huisam.tistory.com/entry/feigntest)
* [Introduction to Spring Cloud OpenFeign](https://www.baeldung.com/spring-cloud-openfeign)
