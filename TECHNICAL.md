# 스프링부트에서 외부 API 테스트하기


### 글을 쓰게 된 계기
![제이슨 퀴즈](./image.png)
<p>

우아한테크코스 코치님이신 제이슨이 여러 크루들 앞에서 재밌는 퀴즈를 냈습니다.
```
지난 미션 중에 토스의 API를 활용하는 부분이 있었죠. 위 그림과 같이 RestClient를 테스트할 수 있을까요?
```
이미지의 상황을 이해해보겠습니다. prod 환경에서 RestCient는 토스 서버를 바라보고 있습니다. 클라이언트-서버 구조에서, 우리의 서버는 클라이언트의 입장이 되고, 토스 서버가 서버의 입장이 됩니다. RestClient가 토스 서버와 요청과 응답을 잘 주고받는지 테스트하고 싶습니다. 하지만 테스트 코드에서 실제로 토스 서버에게 요청을 보내도록 만들면, 우리 서버에 대한 테스트 코드인데 토스 서버의 안정성에 의존해야 한다는 문제가 있습니다. 따라서 application.yml을 적절히 설정하여 test 환경에서는 RestClient가 localhost를 바라보도록 만듭니다. 그리고 FakeTossController라는 테스트용 클래스를 만들어서, RestClient가 localhost로 보내는 요청을 처리할 수 있도록 구성합니다. 자, 이제 테스트가 가능할까요?

</p>
<p>

여기저기 헛다리를 짚는 크루들을 보고 제이슨은 힌트를 줬습니다. 
```
@SpringBootTest의 기본 설정은 무엇인가요?
```
어떤 크루가 대답했습니다.
```
MOCK 입니다!
```
제이슨이 물었습니다.
```
그럼 서블릿 컨테이너가 뜰까요?
```
아! 저는 이 때 서블릿 컨테이너가 무엇인지 몰랐습니다. 그래서 자연스럽게 뒤따라오는 내용은 이해하지도 못했고, 기억도 안 납니다. 분명 이전 과제에서 토스 API를 포함하는 로직에 대해 테스트 코드를 짠 경험이 있는데, 시간이 부족하다는 핑계로 여기저기 블로그에서 돌아다니는 코드와 Chat GPT가 알려준 코드를 어찌저찌 조합해서 제출했던 것으로 기억합니다. 이에 경각심을 느껴 위 주제로 테크니컬라이팅을 시작했습니다. 저와 함께 외부 API를 테스트하는 방법을 고민해보면서, 자연스럽게 서버가 요청을 처리하는 과정까지 이해해봅시다.

</p>

***
### @SpringBootTest의 webEnvironment
<p>

우선 @SpringBootTest가 MOCK 설정일 때, 서블릿 컨테이너가 뜨는가? 라는 질문을 이해해보겠습니다. @SpringBootTest는 다음과 같은 4가지 webEnvironment를 선택할 수 있습니다.
- MOCK
- RANDOM_PORT
- DEFINED_PORT
- NONE

DEFINED_PORT에 대한 설명을 먼저 살펴보겠습니다.

> Loads an EmbeddedWebApplicationContext and provides a real servlet environment. Embedded servlet containers are started and listening on a defined port (i.e from your application.properties or on the default port 8080).

 임베디드 애플리케이션 컨텍스트를 로드해서 실제 서블릿 환경을 제공한다는 게 무슨 뜻일까요? 먼저 스프링부트 애플리케이션을 단순하게 실행시키는 과정을 이해해보겠습니다. 스프링으로 프로젝트를 만들고 main 메서드를 실행킨 뒤, 웹브라우저에서 `http://localhost:8080`이라는 URL로 요청을 보내서 웹브라우저 상에 hello world!를 띄워보신 경험이 있으실 겁니다. main 메서드를 실행시키면 우리의 컴퓨터는 다음과 같은 과정을 거칩니다.
1. 스프링부트 애플리케이션의 내장 톰캣을 하나의 프로세스로 실행시킵니다.
2. 네트워크 요청을 받아야 하는 프로세스이므로, 8080 포트를 할당합니다.

이 때 내장 톰캣이 수행하는 역할 중 하나가 바로 서블릿 컨테이너입니다. 즉 실제 서블릿 환경을 제공한다는 뜻은 내장 톰캣을 띄운다는 말과 큰 차이가 없습니다. 서블릿과 서블릿 컨테이너의 개념이 생소하시다면, 스프링 애플리케이션은 DispatcherServlet이라는 서블릿 하나만이 서블릿 컨테이너에 등록되어 있고, 내장 톰캣으로 들어오는 모든 Http 요청이 DispatcherServlet을 거친다는 점만 이해하셔도 충분합니다. 
![localhost](./localhost.png)
위 그림에서 RestClient가 `http://localhost:8080`으로 요청을 보내면 네트워크를 거쳐 부메랑처럼 요청이 돌아오는 모습을 볼 수 있습니다. 네트워크 입장에서는 웹브라우저가 보내는 요청이든 RestClient가 보내는 요청이든 이들을 localhost:8080이라는 목적지로 전달해준다는 점에서 큰 차이가 없기 때문입니다. 따라서 DEFINED_PORT 옵션을 사용하면 우리가 원하던 FakeController를 사용할 수 있겠습니다. 

***
### RestClient 예제 코드
<p>

FakeController를 활용해서 정말 테스트 코드를 구성할 수 있는지 직접 실험해보겠습니다. RestClient를 사용한 아주 간단한 예제를 아래에서 확인해봅시다.

```java
@Component
public class TodoClient {

    private final RestClient restClient;

    public TodoClient(
            @Value("${todo-base-url}") // application.yml에 정의한 값
            String baseUrl,
            RestClient.Builder builder // RestClient.Builder는 ApplicationContext에 빈으로 등록되어 있어서 생성자 주입 가능
    ) {
        this.restClient = builder.baseUrl(baseUrl).build(); 
    }

    public TodoResponse getTodoById(Long id) {
        return restClient.get()
                .uri("/todos/{id}", id) // baseUrl 뒤에 이 값을 붙여서 최종 URL 완성
                .retrieve()
                .onStatus(status -> status.value() == 404, (req, res) -> {
                    throw new TodoException.NotFound(id);
                })
                .onStatus(status -> status.value() == 500, (req, res) -> {
                    throw new TodoException.InternalServerError(id);
                })
                .body(TodoResponse.class); // 응답으로 받은 JSON을 DTO로 맵핑
    }
}

```

외부 서버에 API 요청을 보냈을 때, 다양한 예외가 발생할 수 있습니다. 사용자가 요청한 자원을 외부 서버에서 찾지 못할 경우(상태코드 404)와 외부 서버에서 내부적인 에러가 발생한 경우(상태코드 500)만 구분해서 커스텀 예외를 던지는 방식으로 간단히 구현했습니다.
```java
public class TodoException extends RuntimeException {
    public TodoException(String message) {
        super(message);
    }

    public static class NotFound extends TodoException {
        public NotFound(Long id) {
            super("Todo not found with id: " + id);
        }
    }

    public static class InternalServerError extends TodoException {
        public InternalServerError(Long id) {
            super("Server error while fetching Todo with id: " + id);
        }
    }
}

```
`${todo-base-url}`은 application.yml에 아래와 같이 적어주었습니다.
```
todo-base-url: http://jsonplaceholder.typicode.com 
```

응답용 DTO는 record를 사용해서 간단하게 구성해봅시다.
```java
public record TodoResponse(long userId, long id, String title, boolean completed) {
}
```
***
### 무엇을 테스트할 것인가?
<p>

`TodoClient는 문제없이 작동합니다!` 라고 주장하기 위해서는 아마 다음과 같은 테스트가 필요할 것입니다.
1. 생성자 주입으로 TodoClient 빈을 생성한다.
2. 외부 서버로부터 API 스펙에 맞게 body를 받으면 TodoResponse라는 DTO로 변환시킨다.
3. 외부 서버로부터 404 상태코드를 응답받으면 TodoException.NotFound 예외를 던진다.
4. 외부 서버로부터 500 상태코드를 응답받으면 TodoException.InternalServerError 예외를 던진다.

TodoClient를 빈으로 등록했기 때문에, 테스트 코드에서 @Autowired를 활용해 주입받을 수 있습니다. 주입받은 TodoClient를 활용해 2번 테스트를 통과시킬 수 있다면, 1번에 대한 테스트도 함께 진행된 것으로 이해할 수 있습니다. 2번 테스트를 위한 환경을 어떻게 구성해야 할까요? 외부 서버에 직접 API 요청을 보내는 방식은 위험합니다. 테스트 코드가 통과하지 못했을 때, 우리 코드의 문제인지, 외부 서버의 문제인지 판단을 어렵게 만들기 때문입니다. 테스트 코드의 목표는 우리 코드가 정상 작동하는지 검증하는 것이므로 외부 서버는 항상 정상 작동한다고 가정하는게 합리적입니다. 항상 정상 작동하는 서버는 어떻게 마련할 수 있을까요?


</p>

***
### @SpringBootTest에서 테스트용 내장 서버를 활용하기
<p>

개인적으로 가장 먼저 떠오른 방법은 테스트용 내장 서버를 8080포트에 띄우는 것입니다. @SpringBootTest의 webEnvironment를 DEFINED_PORT로 설정하면 8080포트로 테스트용 내장 서버를 손쉽게 띄울 수 있습니다. 

```java
@SpringBootTest(webEnvironment = WebEnvironment.DEFINED_PORT) // 8080 포트로 테스트용 내장 서버를 띄웁니다.
public class TodoClientTest {

    @Autowired
    private TodoClient todoClient; // TodoClient는 빈으로 등록되어 있으므로 주입 가능

    @DisplayName("id로 todo를 검색한다.")
    @Test
    void getTodoById() {
        // given
        Long id = 1L;

        // when
        TodoResponse response = todoClient.getTodoById(id);

        // then
        assertAll(
                () -> assertThat(response.userId()).isEqualTo(1L),
                () -> assertThat(response.id()).isEqualTo(1L),
                () -> assertThat(response.title()).isEqualTo("테스트용 타이틀"),
                () -> assertThat(response.completed()).isTrue()
        );
    }
}

```
위 테스트 코드를 통과시키기 위해선 두 가지의 추가 설정이 필요합니다.
1. test 패키지의 application.yml에 아래와 같이 적어줍니다.
```
base-url: http://localhost:8080
```
위와 같은 설정이 필요한 이유는 무엇일까요? 앞서 함께 봤던 TodoClient의 생성자를 다시 봅시다.
```java
@Component
public class TodoClient {

    private final RestClient restClient;

    public TodoClient(
            @Value("${todo-base-url}") // application.yml에 정의한 값
            String baseUrl,
            RestClient.Builder builder // RestClient.Builder는 ApplicationContext에 빈으로 등록되어 있어서 생성자 주입 가능
    ) {
        this.restClient = builder.baseUrl(baseUrl).build(); 
    }

    public TodoResponse getTodoById(Long id) {
        return restClient.get()
                .uri("/todos/{id}", id) // baseUrl 뒤에 이 값을 붙여서 최종 URL 완성
                .retrieve()
                .onStatus(status -> status.value() == 404, (req, res) -> {
                    throw new TodoException.NotFound(id);
                })
                .onStatus(status -> status.value() == 500, (req, res) -> {
                    throw new TodoException.InternalServerError(id);
                })
                .body(TodoResponse.class);
    }
}

```

TodoClient를 빈으로 등록할 때, baseUrl은 application.yml에 정의된 값이 주입됩니다. 스프링부트는 스프링컨테이너를 구성할 때, main의 application.yml과 test의 application.yml을 구분해서 사용합니다. 따라서 테스트용 스프링컨테이너에 TodoClient를 빈으로 등록할 때는 baseUrl이 실제 외부 서버의 URL을 바라보지 않고 localhost:8080을 바라보도록 설정할 수 있습니다. 덕분에 TodoClient는 테스트용 내장 서버를 바라보고 요청을 보낼 수 있게 되었습니다. 예를 들어 todoClient.getTodoById(1)을 호출하면 `GET http://localhost:8080/todos/1`이라는 Http 요청이 전송될 것입니다. 그러나 아직까지 우리의 테스트용 내장 서버는 위 요청을 처리할 컨트롤러가 없습니다. 다음 과정으로 넘어가봅시다.

2. FakeTodoController를 만들어서 빈으로 등록합니다.
```java
@RestController
@RequestMapping("/todos")
public class FakeTodoController {

    @GetMapping("/{id}")
    public Map<String, String> getById(@PathVariable Long id) {
        Map<String, String> map = new HashMap<>();
        map.put("userId", "1");
        map.put("id", "1");
        map.put("title", "테스트용 타이틀");
        map.put("completed", "true");
        return map;
    }
}
```

FakeTodoController는 Map을 반환합니다. @RestController에 포함되어 있는 @ResponseBody 덕분에 Map은 JSON 형태로 변환되어 HttpResponse의 body에 담기게 됩니다. 이 때 Map이 아니라 앞서 만들어둔 TodoResponse라는 DTO를 사용하면 어떨까요? RestClient에게 보내줄 HttpResponse의 body값을 준비하는 과정에서 DTO를 JSON으로 스프링이 알아서 변환해줄 것입니다. 만약 DTO의 필드명이나 타입이 API 스펙과 다르게 적혀있었다면 어떻게 될까요? 실제 운영환경에서는 문제가 발생함에도 불구하고 테스트 코드는 통과할 것입니다. 서버가 실제 보내줄 JSON 값이 우리의 DTO와 항상 대응할 것이라 가정하고 FakeController를 작성했기 때문입니다. 따라서 외부 서버가 API 응답으로 보내주는 스펙에 맞게 FakeController에서 반환하는 Map의 key, value값을 정확하게 기입해주는 것도 굉장히 중요한 작업입니다.

</p> 

<p>

긴 호흡으로 설명드렸지만, 사실 위 방법은 테스트용 내장 서버를 실제로 띄워야 한다는 사실 자체에서 큰 단점을 안고 있습니다. 테스트 코드를 실제로 실행해보면 테스트용 내장 서버를 띄운 경우, 그렇지 않은 경우보다 훨씬 오래 걸리기 때문입니다. 만약 매 테스트 메서드마다 애플리케이션 컨텍스트를 새로 생성하는 @DirtiesContext 옵션까지 사용한다면, 애플리케이션 컨텍스트가 새로 생성될 때마다 내장 서버도 새로 띄워야 하기에 엄청나게 오래 걸리는 테스트 코드가 완성됩니다. 따라서 내장 톰캣을 띄우지 않고 테스트하는 방법이 있다면, 이를 선택하는 것이 합리적입니다.

</p>
***

### @RestClientTest
<p>

스프링에서는 내장 톰캣을 띄우지 않고도 외부 API 요청을 테스트할 수 있도록 @RestClientTest를 제공합니다. RestClient가 보내는 HttpRequest를 MockRestServiceServer라는 Mock서버가 가로채서 대신 응답해주는 방식으로 작동합니다. Mock서버는 실제 서버가 아니므로 어떤 요청을 받을 지, 그리고 요청에 대해 어떤 응답을 내보낼 지에 대해 직접 정의해줘야 합니다. 아래의 코드를 주석과 함께 확인해보시면 쉽게 이해하실 수 있습니다.

```java
// value에 테스트 대상을 등록합니다. 단, RestTemplateBuilder 또는 RestClient.Builder에 의존하는 빈만 가능합니다.
@RestClientTest(value = TodoClient.class) 
class TodoClientTest {

    @Autowired
    private TodoClient todoClient; // 위에서 빈으로 지정해줬기 때문에 주입 가능

    @Autowired
    private MockRestServiceServer mockServer;

    @BeforeEach
    void setUp() {
        String expectedResult = // JSON 응답을 String 형태로 만들 수 있어서 가독성이 좋다.
                """
                        {
                          "userId": 1,
                          "id": 1,
                          "title": "테스트용 타이틀",
                          "completed": true
                        }
                        """;
        mockServer.expect(requestTo("http://localhost:8080/todos/1")) // mockServer가 받을 수 있는 요청을 정의
                .andExpect(method(HttpMethod.GET)) // MockRestRequestMatchers.method를 import 해야함에 주의!
                .andRespond(withSuccess(expectedResult, MediaType.APPLICATION_JSON)); // 요청에 대해 내보내는 응답을 정의
    }

    @DisplayName("id로 todo를 검색한다.")
    @Test
    void getTodoById() {
        // given
        Long id = 1L;

        // when
        TodoResponse response = todoClient.getTodoById(id);

        // then
        assertAll(
                () -> assertThat(response.userId()).isEqualTo(1L),
                () -> assertThat(response.id()).isEqualTo(1L),
                () -> assertThat(response.title()).isEqualTo("테스트용 타이틀"),
                () -> assertThat(response.completed()).isTrue()
        );
    }
}

```

코드에서 확인하신 것처럼 @RestClientTest를 MockRestServiceServer와 함께 사용하면, 실제 서버를 띄울 필요도 없고 FakeTodoController와 같은 테스트용 컨트롤러도 필요하지 않습니다. 애초에 스프링에서 REST API client에 대한 테스트를 하라고 만들어 놓은 환경이기 때문에 사용하지 않을 이유가 없습니다.

</p>
<p>
@RestClientTest는 REST API client에 대한 테스트에서 꼭 필요한 빈들만 등록된 컨테이너를 사용합니다. 따라서 일반적인 @SpringBootTest에서 사용하는 애플리케이션 컨텍스트보다 경량화된 컨테이너를 사용하므로, 애플리케이션 컨텍스트를 생성하는 속도가 비교적 빠릅니다. 참고로 @RestClientTest를 정의한 클래스에 적힌 주석을 읽어보면, 이 어노테이션을 사용할 때 활용할 수 있는 빈들은 다음과 같습니다.

1. 테스트 대상으로 등록한 빈
2. @JsonComponent 어노테이션이 달린 빈
3. Jackson 라이브러리가 사용 가능한 경우, Jackson 모듈을 구현한 빈
4. MockRestServiceServer
즉, 대부분의 빈은 컨테이너에 등록되지 않습니다.

</p>

***

### 참고 문헌
- [TodoClient 코드 출처](https://github.com/cho-log/spring-learning-test/blob/main/spring-http-client-1/complete/src/main/java/cholog/TodoClientWithRestClient.java) 
- [SpringBootTest의 webEnvironment](https://docs.spring.io/spring-boot/docs/1.5.2.RELEASE/reference/html/boot-features-testing.html)
- [여러 개의 RestClient를 사용할 때 주의점](https://github.com/spring-projects/spring-boot/issues/38820)