## BookService 구현


**BookService** 같은 경우는 해당 스터디원이 완성을 못하게 되면서 필자가 맡아서 완성을 시켰다.
기본적인 클래스는 코드만 남기겠다.

### Entity

```java
package com.study.Book.Service.entity;

import lombok.Getter;
import lombok.Setter;

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
@Getter
@Setter
public class Book {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private long id;
    private String name;
    private String author;
    private String genre;
    private int stock;
}
```

### Repository

```java
package com.study.Book.Service.repository;

import com.study.Book.Service.entity.Book;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

public interface BookRepository extends JpaRepository<Book, Long> {
    @Query("SELECT b.stock FROM Book b WHERE b.id = :bookId")
    int findStockById(long bookId);
}
```

### Feign

```java
package com.study.Book.Service.feign;

import com.study.Book.Service.response.BookStockResponse;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;

@FeignClient(name = "HistoryFeign", url = "http://localhost:8083/api")
public interface HistoryFeign {
    @GetMapping("/histories/book/{bookId}")
    BookStockResponse getBorrowedBookStock(@PathVariable("bookId") long bookId);
}
```

### Response

```java
package com.study.Book.Service.response;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class BookStockRecord {
    private long userId;
    private int quantity;
}
```

```java
package com.study.Book.Service.response;

import lombok.Getter;
import lombok.NoArgsConstructor;

import java.util.List;

@Getter
@NoArgsConstructor
public class BookStockResponse {
    private List<BookStockRecord> bookStockRecords;
}
```

<img width="702" alt="스크린샷 2023-07-31 오후 5 42 25" src="https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/ef598726-795f-4143-870d-d62a8501d95f">

HistoryFeign에서 위 그림의 형식으로 보내주기 때문에 응답 값에 맞춰서 만들었다.

아래는 코드는 **BookService가 Loan에게 보내는 응답 형태**이다.

```java
package com.study.Book.Service.response;

public enum BookAvailabilityStatus {
    AVAILABLE,
    NOT_AVAILABLE
}
```

### BookService

```java
package com.study.Book.Service.service;

import com.study.Book.Service.response.BookAvailabilityStatus;
import com.study.Book.Service.entity.Book;
import com.study.Book.Service.feign.HistoryFeign;
import com.study.Book.Service.repository.BookRepository;
import com.study.Book.Service.response.BookStockRecord;
import com.study.Book.Service.response.BookStockResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;

import javax.persistence.EntityNotFoundException;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class BookService {
    private final BookRepository bookRepository;
    private final HistoryFeign historyFeign;

    public BookAvailabilityStatus checkBookAvailabilityForRental(long bookId) {
        int totalStock = getTotalBookStock(bookId);
        BookStockResponse stockResponse = historyFeign.getBorrowedBookStock(bookId);
        List<BookStockRecord> stockRecords = stockResponse.getBookStockRecords();
        int rentedBook = stockRecords.stream()
                .mapToInt(BookStockRecord::getQuantity)
                .sum();

        if (totalStock - rentedBook <= 0) return BookAvailabilityStatus.NOT_AVAILABLE;
        return BookAvailabilityStatus.AVAILABLE;
    }

    public int getTotalBookStock(long bookId) {
        Book book = bookRepository.findById(bookId)
                .orElseThrow(() -> new EntityNotFoundException("ID가 " + bookId + "인 도서를 찾을 수 없습니다."));
        return book.getStock();
    }
}
```

우선 `getTotalBookStock()` 메서드부터 설명하겠다. 해당 메서드는 **bookRepository에서 bookId를 찾아 stock을 return**해준다. 만약 해당 **bookId가 없는 경우에는 예외**를 던진다.
이 메서드는 `checkBookAvailabilityForRental()` 메서드에서 쓰인다.

이제 `checkBookAvailabilityForRental()` 를 설명하겠다.
`getTotalBookStock()` 에서 가져온 stock을 **totalStock**에 넣어주고, `historyFeign` 에서 받아온 데이터를 *`List*<BookStockRecord>` 형태의 **stockRecords**에 넣어준 뒤 stream으로 quantity값을 더해서 **rentedBook**에 넣준다.
그리고 만약 **totalStock - rentedBook 값이 0보다 같거나 작으면** 대여를 못한다는 **NOT_AVAILABLE**을 반환하고, 그게 아니면 여분이 있는 것으로 **AVAILABLE**을 반환해준다.

### BookServiceTest

```java
package com.study.Book.Service;

import com.study.Book.Service.entity.Book;
import com.study.Book.Service.feign.HistoryFeign;
import com.study.Book.Service.repository.BookRepository;
import com.study.Book.Service.response.BookAvailabilityStatus;
import com.study.Book.Service.response.BookStockRecord;
import com.study.Book.Service.response.BookStockResponse;
import com.study.Book.Service.service.BookService;
import org.junit.jupiter.api.DisplayName;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import javax.persistence.EntityNotFoundException;
import java.util.Collections;
import java.util.List;
import java.util.Optional;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class BookServiceTest {
    @InjectMocks
    BookService bookService;
    @Mock
    BookRepository bookRepository;
    @Mock
    HistoryFeign historyFeign;

    @Test
    @DisplayName("책 대여가 가능한 경우")
    public void testBookAvailable() {
        // given
        long bookId = 1L;
        int stock = 5;
        int quantity = 2;

        Book book = new Book();
        book.setId(bookId);
        book.setStock(stock);

        List<BookStockRecord> stockRecords = Collections.singletonList(new BookStockRecord(1L, quantity));

        BookStockResponse stockResponse = mock(BookStockResponse.class);
        when(stockResponse.getBookStockRecords()).thenReturn(stockRecords);
        when(historyFeign.getBorrowedBookStock(bookId)).thenReturn(stockResponse);
        when(bookRepository.findById(bookId)).thenReturn(java.util.Optional.of(book));
        // when
        BookAvailabilityStatus result = bookService.checkBookAvailabilityForRental(bookId);
        // then
        assertEquals(BookAvailabilityStatus.AVAILABLE, result);
        verify(bookRepository).findById(bookId);
        verify(historyFeign).getBorrowedBookStock(bookId);
    }

    @Test
    @DisplayName("책 대여가 불가능한 경우")
    public void testBookNotAvailable() {
        // given
        long bookId = 1L;
        int stock = 5;
        int quantity = 5;

        Book book = new Book();
        book.setId(bookId);
        book.setStock(stock);

        List<BookStockRecord> stockRecords = Collections.singletonList(new BookStockRecord(1L, quantity));

        BookStockResponse stockResponse = mock(BookStockResponse.class);
        when(stockResponse.getBookStockRecords()).thenReturn(stockRecords);
        when(historyFeign.getBorrowedBookStock(bookId)).thenReturn(stockResponse);
        when(bookRepository.findById(bookId)).thenReturn(Optional.of(book));
        // when
        BookAvailabilityStatus result = bookService.checkBookAvailabilityForRental(bookId);
        // then
        assertEquals(BookAvailabilityStatus.NOT_AVAILABLE, result);
        verify(bookRepository).findById(bookId);
        verify(historyFeign).getBorrowedBookStock(bookId);
    }

    @Test
    @DisplayName("책의 재고를 반환하는 경우")
    public void testGetBookStock() {
        // given
        long bookId = 1L;
        int stock = 5;

        Book book = new Book();
        book.setId(bookId);
        book.setStock(stock);

        when(bookRepository.findById(bookId)).thenReturn(Optional.of(book));
        //when
        int bookStock = bookService.getTotalBookStock(bookId);
        //then
        assertEquals(stock, bookStock);
        verify(bookRepository).findById(bookId);
    }

    @Test
    @DisplayName("bookId를 못 찾는 경우 예외를 던지는지 확인")
    public void testBookStockNotFound() {
        // given
        long nonExistentBookId = 9L;
        when(bookRepository.findById(nonExistentBookId)).thenReturn(Optional.empty());
        // when, then
        assertThrows(EntityNotFoundException.class, () -> {
            bookService.getTotalBookStock(nonExistentBookId);
        });
        verify(bookRepository).findById(nonExistentBookId);
    }
}
```

우선 앞서 동일하게 적용한 코드들은 넘어가고, 새롭게 작성한 코드들을 설명하겠다.
먼저 아래 코드부터 보자.

```java
List<BookStockRecord> stockRecords = Collections.singletonList(new BookStockRecord(1L, quantity));
```

이 코드는 **stockRecords**라는 리스트를 생성하고, 그안에 **BookStockRecord** 객체를 하나만 포함시키기위해 `Collections.singletonList()`를 사용했다.

그다음 코드를 보자.

```java
BookStockResponse stockResponse = mock(BookStockResponse.class);
```

`mock(BookStockResponse.class)` 는 **BookStockResponse** 클래스의 **mock 객체를 생성**하는 메서드다.
이렇게 작성하면 **stockResponse**라는 **mock 객체를 생성**한다.
`stockResponse.getBookStockRecords()`메서드를 호출하면 **stockRecords 리스트를 반환**한다.

아래 코드를 보자.

```java
when(bookRepository.findById(bookId)).thenReturn(Optional.of(book));
```

이 코드는 **bookRepository**의 `findById()`메서드가 호출될때 **bookId**에 해당하는 책이 **데이터베이스에 존재한다면 해당 책을 담은 Optional 객체를 반환**하도록 작성했다. 이렇게 하면 **테스트 코드**에서는 데이터베이스의 책 정보를 **미리 설정한 book 객체로 대체하여 테스트를 수행**할 수 있다.
