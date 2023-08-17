## Mock Service 구현(Test 역할)

테스트용이기 때문에 임의의 값을 넣어두고 코드를 작성해야 했다.
아래 그림을 보면 각 Member Service와 Book Service에게 어떤 값을 넘겨줄지 예상할 수 있다.

![스크린샷 2023-07-03 오후 5 22 22](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/23fed857-0e38-4475-8229-0ff18a5b2f3a)

port 번호는 테스트용으로 2개의 서버를 돌리며 테스트해야하기때문에 기존 8080과 다르게 바꿔주었다. 
각 서비스에 quantity 값을 넘겨주는 이유는 요구사항을 봤으면 알겠지만, **대여 수와 책 여분의 수를 가지고 대여의 가능 여부를 판단**하기 때문이다.
우선 application.properties에 **임의의 값을 넣어주는 법**은 간단하다. 그냥 **이름을 만들고 ‘=’ 에 원하는 값을 넣어**주면 된다.  

### Controller 구현

아래 코드를 보면

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api")
public class TestController {
    private final TestService testService;

    @GetMapping("/test/member/{memberId}")
    public ApiResponse<Map<String, Object>> getMemberQuantity(@PathVariable long memberId) {
        return ApiResponse.ok(testService.getRentalQuantity(memberId));
    }

    @GetMapping("/test/book/{bookId}")
    public ApiResponse<Map<String, Object>> getBookQuantity(@PathVariable long bookId) {
        return ApiResponse.ok(testService.getBookQuantity(bookId));
    }
}
```

각 **Get 요청**에 `"/test/member/{memberId}", "/test/book/{bookId}”`에 **맞는 Id값을 넣으면** 각 **testService에 있는 해당 메서드에서 값을 가져온다.**

그럼 TestService를 봐보자

### Service 구현

```java
@Service
public class TestService {
    @Value("${memberId}")
    private long memberId;
    @Value("${quantity}")
    private int quantity;
    @Value("${bookIds}")
    private List<Long> bookIds;
    @Value("${bookId}")
    private Long bookId;
    @Value("${stock}")
    private int stock;

    public Map<String, Object> getRentalQuantity(long memberId) {
        if (this.memberId != memberId) throw new TestException(TestRtnConsts.ERR400);
        Map<String, Object> response = new LinkedHashMap<>();
        response.put("quantity", this.quantity);
        response.put("bookId", bookIds);
        return response;
        }

    public Map<String, Object> getBookQuantity(long bookId) {
        if (this.bookId != bookId) throw new TestException(TestRtnConsts.ERR401);
        Map<String, Object> response = new LinkedHashMap<>();
        response.put("bookId", this.bookId);
        response.put("stock", this.stock);
        return response;
        }
    }
```

우선 **@Value** 어노테이션에 대해서 설명하겠다.
**@Value**은 Spring 프레임워크의 일부이며 **외부 소스의 값을 Spring 관리 빈에 주입**하는 데 사용된다.
**속성 파일, 환경 변수, 명령줄** 인수 또는 **시스템 속성**과 같은 다양한 소스에서 값을 검색하고 **Bean 내의 필드 또는 생성자 인수에 할당**할 수 있다.

정리하자면
**@Value**를 이용해 외부(`properties`)에 있는 값을 주입받아 필드에 할당함으로써, TestService에서 이 값을 사용하는 것이다.
해당 테스트 사용자가 Id가 입력한 값이랑 맞을 경우 미리 설정해 놓은 각 quantity 값이 전달된다.
틀릴 경우 `TestException(TestRtnConsts.설정해 놓은 메시지)` 가 전달된다.
