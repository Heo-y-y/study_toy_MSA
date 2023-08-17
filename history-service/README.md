## HistoryService 구현


### 사전 준비

**HistoryService**는 기록을 userService나 bookService에게 보내주는 **저장소 역할**이다.
그래서 먼저 보내줄 데이터를 Repository에 넣어 줄 필요가 있었다.
**mySQL**을 사용하고 있으므로 우선 resources 패키지에 데이터를 넣기 위해 **data-init.sql** 파일을 만들었다.

![스크린샷 2023-07-16 오후 6 44 31](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/daa10764-b24f-481c-8e51-4e73dd6224cc)

그리고 history 테이블 안에 컬럼 값들을 미리 넣어줬다.

![스크린샷 2023-07-16 오후 6 51 31](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/2cc3de85-d1f7-42db-ad5a-404e59b84329)

userService를 담당하신 분이 요청 응답을 아래 형식으로 받기를 원했다.

```json
{
    "userId 또는 bookId 기록": [
        {
            "userId 또는 bookId": long,
            "quantity": int
        },
        {
            "userId 또는 bookId": long,
            "quantity": int
        }
    ]
}
```

위 응답 형식을 참고하여 userService와 bookService에게 보내주기로 했다.

### History

```java
package com.study.history.service.domain;

import lombok.Getter;
import lombok.NoArgsConstructor;
import javax.persistence.*;

@Getter
@Entity
@NoArgsConstructor
public class History {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(nullable = false, unique = true)
    private long id;
    @Column(nullable = false)
    private long userId;
    @Column(nullable = false)
    private long bookId;
    @Column(nullable = false)
    private int quantity;
		}
}
```

### HistoryRepository

```java
package com.study.history.service.repository;

import com.study.history.service.domain.History;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface HistoryRepository extends JpaRepository<History, Long> {
    List<History> findAllByUserId(long userId);

    List<History> findAllByBookId(long bookId);
}
```

요청하는 기록을 해당 **userId**로 찾는 메서드와 해당 **bookId**로 찾는 메서드를 만들었다.

### RenatalRecord(userService에게 보내는 response)

```java
package com.study.history.service.response;

import com.study.history.service.domain.History;
import lombok.AccessLevel;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class RentalRecord {
    private long bookId;
    private int quantity;

    public static RentalRecord of(History history) {
        return new RentalRecord(
                history.getBookId(),
                history.getQuantity());
    }
}
```

먼저 **userService**에게 보내는 응답 dto를 만들었다.

### RentalResponse(userService에게 보내는 List response)

```java
package com.study.history.service.response;

import com.study.history.service.domain.History;
import lombok.AccessLevel;
import lombok.AllArgsConstructor;
import lombok.Getter;
import java.util.List;
import java.util.stream.Collectors;

@Getter
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class RentalResponse {
    private List<RentalRecord> userRentalRecords;

    public static RentalResponse of(List<History> userRentalRecords) {
        return new RentalResponse(userRentalRecords.stream().map(RentalRecord::of).collect(Collectors.toList()));
    }
}
```

RentalRecord를 static class로 만들 수도 있었지만, 필자는 그런 방식을 선호하지 않아서 따로 만들어서 사용하는 형식으로 작성했다.

### StockRecord(bookService에게 보내는 response)

```java
package com.study.history.service.response;

import com.study.history.service.domain.History;
import lombok.AccessLevel;
import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class StockRecord {
    private long userId;
    private int quantity;

    public static StockRecord of(History history) {
        return new StockRecord(
                history.getUserId(),
                history.getQuantity()
        );
    }
}
```

### StockResponse(bookService에게 보내는 List response)

```java
package com.study.history.service.response;

import com.study.history.service.domain.History;
import lombok.AccessLevel;
import lombok.AllArgsConstructor;
import lombok.Getter;
import java.util.List;
import java.util.stream.Collectors;

@Getter
@AllArgsConstructor(access = AccessLevel.PRIVATE)
public class StockResponse {
    private List<StockRecord> bookStockRecords;

    public static StockResponse of(List<History> bookStockRecords) {
        return new StockResponse(bookStockRecords.stream().map(StockRecord::of).collect(Collectors.toList()));
    }
}
```

### HistoryService

```java
package com.study.history.service.service;

import com.study.history.service.response.RentalResponse;
import com.study.history.service.repository.HistoryRepository;
import com.study.history.service.response.StockResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import javax.persistence.EntityNotFoundException;

@Service
@Slf4j
@RequiredArgsConstructor
public class HistoryService {
    private final HistoryRepository historyRepository;

    public RentalResponse getRentalRecords(long userId) {
        if (historyRepository.findAllByUserId(userId).isEmpty()) {
            EntityNotFoundException userEntityNotFoundException = new EntityNotFoundException("The User ID " + userId + " could not be found.");
            log.error("userId 처리 중 error 발생한 userId: " + userId, userEntityNotFoundException);
            throw new EntityNotFoundException("해당 User Id를 찾을 수 없습니다.");
        }
        return RentalResponse.of(historyRepository.findAllByUserId(userId));
    }

    public StockResponse getStockRecords(long bookId) {
        if (historyRepository.findAllByBookId(bookId).isEmpty()) {
            EntityNotFoundException bookEntityNotFoundException = new EntityNotFoundException("The Book ID" + bookId + " could not be found");
            log.error("bookId 처리 중 error 발생한 bookId: " + bookId, bookEntityNotFoundException);
            throw new EntityNotFoundException("해당 Book Id를 찾을 수 없습니다.");
        }
        return StockResponse.of(historyRepository.findAllByBookId(bookId));
    }
}
```

우선 해당 **userId** 또는 **bookId**가 **없을 경우** `EntityNotFoundException` 를 던지게 끔하고 따로 **로그**로 남겨 볼 수 있는 코드랑 스터디원이 조금 더 편하게 진행하도록 **응답 메시지**를 보내는 코드도 작성했다.
MockTest에서는 따로 커스텀 ApiResponse를 만들었지만 **HistoryService**에서는 **스터디원과 따로 사전에 정한 규칙이 없고, 괜한 오해가 생길 우려가 있어 제공되는 예외를 활용**했다.
역기서 **Service 클래스의 역할**은 해당 **userId와 bookId가 같은지 필터링**해주는 작업을 한다.

### HistroyController

```java
package com.study.history.service.controller;

import com.study.history.service.response.RentalResponse;
import com.study.history.service.response.StockResponse;
import com.study.history.service.service.HistoryService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
@RequiredArgsConstructor
public class HistoryController {
    private final HistoryService historyService;

    @GetMapping("/histories/user/{userId}")
    public ResponseEntity<RentalResponse> sendRentalRecords(@PathVariable long userId) {
        RentalResponse rentalRecords = historyService.getRentalRecords(userId);
        return ResponseEntity.ok(rentalRecords);
    }

    @GetMapping("/histories/book/{bookId}")
    public ResponseEntity<StockResponse> sendStockRecords(@PathVariable long bookId) {
        StockResponse stockRecords = historyService.getStockRecords(bookId);
        return ResponseEntity.ok(stockRecords);
    }
}
```

**컨트롤러의 역할**은 간단하다 레파지토리에서 가지고온 데이터를 서비스에서 해당 userId 또는 bookId와 같은지 필터링 후 그냥 가져오는 역할이다.

### RestExceptionHandler(예외처리)

```java
package com.study.history.service.response.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import javax.persistence.EntityNotFoundException;

@RestControllerAdvice
public class RestExceptionHandler {
    @ExceptionHandler
    public ResponseEntity<String> handleUserEntityNotFoundException(EntityNotFoundException ex) {
        return new ResponseEntity<>(ex.getMessage(), HttpStatus.NOT_FOUND);
    }
}
```

따로 **메시지를 응답바디에 전달**하고 싶어서 **RestExceptionHandler**를 구현했다.
처음에는 **NullPointerException**도 같이 적용했는데 위에 History에서 `@Column(nullable = false)`를 사용하여 필요성을 느끼지 못해 삭제했다.
**HistoryService**는 **userId**와 **bookId**를 이용해 **해당 기록들을 보여주는 API를 제공**한다. 그래서 해당 Id가 없을 경우의 예외처리를 `EntityNotFoundException`를 사용하여 적용시켰다.

## Test Code 작성

### HistoryRepositoryTest

```java
package com.study.history.service.repository;

import com.study.history.service.domain.History;
import com.study.history.service.response.repository.HistoryRepository;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase;
import org.springframework.boot.test.autoconfigure.orm.jpa.DataJpaTest;

import java.util.List;

import static org.assertj.core.api.Assertions.assertThat;
import static org.springframework.boot.test.autoconfigure.jdbc.AutoConfigureTestDatabase.*;

@DataJpaTest
@AutoConfigureTestDatabase(replace = Replace.NONE)
class HistoryRepositoryTest {
    @Autowired
    private HistoryRepository historyRepository;

    @Test
    @DisplayName("저장된 userId 기록이 제대로 조회되는지 확인")
    void findAllByUserIdTest() {
        // when
        List<History> userRentalRecords = historyRepository.findAllByUserId(2L);
        // then
        assertThat(userRentalRecords).isNotNull();
        assertThat(userRentalRecords).hasSize(2);
        assertThat(userRentalRecords.get(0).getBookId()).isEqualTo(2L);
        assertThat(userRentalRecords.get(0).getQuantity()).isEqualTo(1);
        assertThat(userRentalRecords.get(1).getBookId()).isEqualTo(1L);
        assertThat(userRentalRecords.get(1).getQuantity()).isEqualTo(1);
    }

    @Test
    @DisplayName("저장된 bookId 기록이 제대로 조회되는지 확인")
    void findAllByBookIdTest() {
        // when
        List<History> bookStockRecords = historyRepository.findAllByBookId(1L);
        // then
        assertThat(bookStockRecords).isNotNull();
        assertThat(bookStockRecords).hasSize(2);
        assertThat(bookStockRecords.get(0).getUserId()).isEqualTo(1L);
        assertThat(bookStockRecords.get(0).getQuantity()).isEqualTo(1);
        assertThat(bookStockRecords.get(1).getUserId()).isEqualTo(2L);
        assertThat(bookStockRecords.get(1).getQuantity()).isEqualTo(1);
    }

    @Test
    @DisplayName("데이터가 DB에 저장이 잘 되는지 확인")
    void saveHistoryTest() {
        // given
        History history = new History(3L, 5L, 1);
        // When
        History savedHistory = historyRepository.save(history);
        // then
        assertThat(savedHistory).isNotNull();
        assertThat(savedHistory.getUserId()).isEqualTo(3L);
        assertThat(savedHistory.getBookId()).isEqualTo(5L);
        assertThat(savedHistory.getQuantity()).isEqualTo(1);
    }

    @Test
    @DisplayName("여러 데이터를 한번에 저장할 경우 DB에 저장이 잘 되는지 확인")
    void SaveAllHistoryTest() {
        // given
        List<History> histories = List.of(
                new History(3L, 1L, 1),
                new History(4L, 4L, 2)
        );
        // when
        List<History> savedHistories = historyRepository.saveAll(histories);
        // then
        assertThat(savedHistories)
                .isNotNull()
                .hasSameSizeAs(histories)
                .containsExactlyElementsOf(histories);
    }
}
```

필자는 **data-init.sql**에 있는 데이터를 활용해서 테스트하고 싶었다.
평소에는 `@ExtendWith(MockitoExtension.*class*)` 를 사용했었는데 레파지토리를 테스트할 때는 `@DataJpaTest` 를 사용했다.
`@DataJpaTest` 는 Spring 애플리케이션 컨텍스트를 설정할 필요 없이 저장소 인터페이스와 기본 데이터베이스 간의 상호 작용을 테스트하는 데 집중할 수 있다고 한다.

처음에는 `@DataJpaTest` 만 사용하니까 계속해서 에러가 나왔다. 
그래서 `@AutoConfigureTestDatabase(replace = Replace.NONE)` 를 사용했더니 해결됐다.
자세한 이유는 **[ErrorLog](https://github.com/Heo-y-y/study_toy_MSA/blob/main/history-service/error-log.md)** 에서 확인하기 바란다.

레파지토리 테스트는 이 정도만 알면 쉽게 테스트가 가능했다.

### HistoryServiceTest

```java
package com.study.history.service.service;

import com.study.history.service.domain.History;
import com.study.history.service.response.repository.HistoryRepository;
import com.study.history.service.response.RentalRecord;
import com.study.history.service.response.RentalResponse;
import com.study.history.service.response.StockRecord;
import com.study.history.service.response.StockResponse;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.ArrayList;
import java.util.List;
import java.util.stream.Collectors;

import static org.assertj.core.api.AssertionsForInterfaceTypes.assertThat;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class HistoryServiceTest {
    @Mock
    HistoryRepository historyRepository;
    @InjectMocks
    HistoryService historyService;

    @Test
    @DisplayName("getRentalRecords() 테스트")
    void getRentalRecordsTest() {
        // given
        long userId = 1L;
        List<History> testDataList = new ArrayList<>();
        testDataList.add(new History(1, 1, 2));
        testDataList.add(new History(1, 2, 2));
        testDataList.add(new History(2, 3, 1));
        testDataList.add(new History(3, 4, 1));
        when(historyRepository.findAllByUserId(userId)).thenReturn(
                testDataList.stream()
                        .filter(history -> history.getUserId() == userId)
                        .collect(Collectors.toList())
        );
        // when
        RentalResponse rentalResponse = historyService.getRentalRecords(userId);
        // then
        assertThat(rentalResponse).isNotNull();
        List<RentalRecord> rentalRecords = rentalResponse.getUserRentalRecords();
        assertThat(rentalRecords).hasSize(2);
        assertThat(rentalRecords.get(0).getBookId()).isEqualTo(1L);
        assertThat(rentalRecords.get(0).getQuantity()).isEqualTo(2);
        assertThat(rentalRecords.get(1).getBookId()).isEqualTo(2L);
        assertThat(rentalRecords.get(1).getQuantity()).isEqualTo(2);
    }

    @Test
    @DisplayName("getStockRecords() 테스트")
    void getStockRecordsTest() {
        // given
        long bookId = 2L;
        List<History> testDataList = new ArrayList<>();
        testDataList.add(new History(2, 2, 1));
        testDataList.add(new History(1, 2, 2));
        testDataList.add(new History(3, 1, 1));
        testDataList.add(new History(2, 3, 1));
        when(historyRepository.findAllByBookId(bookId)).thenReturn(
                testDataList.stream().filter(history -> history.getBookId() == bookId)
                        .collect(Collectors.toList())
        );
        // when
        StockResponse stockResponse = historyService.getStockRecords(bookId);
        // then
        assertThat(stockResponse ).isNotNull();
        List<StockRecord> stockRecords = stockResponse.getBookStockRecords();
        assertThat(stockRecords).hasSize(2);
        assertThat(stockRecords.get(0).getUserId()).isEqualTo(2L);
        assertThat(stockRecords.get(0).getQuantity()).isEqualTo(1);
        assertThat(stockRecords.get(1).getUserId()).isEqualTo(1L);
        assertThat(stockRecords.get(1).getQuantity()).isEqualTo(2);
    }
}
```

**Service** 클래스 테스트는 **필터링 처리가 제대로 작동하는지를 테스트**했다.
진행하기에 앞서 먼저 **레파지토리와 분리**하는게 필요해 **실제 구현에 의존하지 않고 동작을 제어하기위해** `@Mock`을 **레파지토리**에 적용하여 **가짜 객체를 service에 주입**받게 하기 위해 `@InjectMocks` 를 사용했다.
