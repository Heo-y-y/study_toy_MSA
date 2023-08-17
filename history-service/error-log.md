# **org.springframework.beans.factory.UnsatisfiedDependencyException 에러**


MSA 학습중 레파지토리를 처음 테스트하게 됐다.
테스트하는 도중에 타이틀의 에러를 접하게 됐다. 
우선 상황은 레파지토리를 테스트하기위해 Spring Boot Test에서 제공하는 `@DataJpaTest` 를 사용하는데, 필자는 data-init.sql이라는 파일에 미리 적용한 데이터를 가지고 테스트하고 싶었다.

![스크린샷 2023-07-17 오후 10 13 32](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/d21e2cb6-08d2-4a8a-8700-aa27f1f507c3)

그래서 코드를 작성하고 테스트를 돌리니 아래의 에러를 만났다.

![스크린샷 2023-07-17 오후 10 16 25](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/bbf4bac0-4c30-454b-94ba-4c114b1fa78f)

필자는 영어를 잘하지 못한다. 그래서 해당 `UnsatisfiedDependencyException` 에 들어가보니 이렇게 나왔다.

![스크린샷 2023-07-17 오후 10 17 50](https://github.com/heo-mewluee-Study-Group/cs-study/assets/112863029/17df938f-4663-40fb-8b7f-215132e6e2a2)
그래서 해당 문서를 번역해 보니
**종속성 검사가 활성화되었지만 bean이 다른 bean 또는 bean factory 정의에 지정되지 않은 단순 속성에 종속될 때 발생하는 예외입니다.**
라는 내용이였다. 이게 무슨 소리지….?

왜그러는지 구글링의 검색을 통해 알게 된 이유는 `@DataJpaTest` **를 사용할 때 구성된 테스트용 데이터베이스가 없어서 발생**하는 것이란다.
구체적인 오류 메세지는 스프링 부트가 테스트 목적으로 데이터 소스를 내장된 데이터베이스로 대체하지 못했음을 의미한다.
발생하는 이유는 **내장된 데이터베이스나 지원되는 데이터베이스 종속성이 클래스패스에 없기 때문**이라고 한다.

내장된 데이터베이스나 적합한 대체 요소가 없는 경우, 스프링 부트는 테스트용 데이터베이스를 처리하기 위해 **명시적인 구성이 필요**한데 이때 사용하는 것이 `@AutoConfigureTestDatabase(replace = Replace.NONE)` 이다. `@AutoConfigureTestDatabase(replace = Replace.NONE)` 를 사용하는 이유는 데이터 소스를 대체하지 않고, **기존의 구성된 데이터베이스를 테스트에 사용**하도록 스프링부트에 지시를 해주기 때문이다.

### TestCode
``` java
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
//    @DisplayName("저장된 bookId 기록이 제대로 조회되는지 확인")
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
