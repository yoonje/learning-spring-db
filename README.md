# Spring Framework 정리 자료
스프링 DB 1편 - 데이터 접근 핵심 원리

Table of contents
=================
<!--ts-->
   * [JDBC 이해](#JDBC-이해)
   * [커넥션풀과 데이터소스 이해](#커넥션풀과-데이터소스-이해)
   * [트랜잭션 이해](#트랜잭션-이해)
   * [스프링과 트랜잭션 문제 해결](#스프링과-트랜잭션-문제-해결)
   * [자바 예외 이해](#자바-예외-이해)
   * [스프링과 예외 문제 해결](#스프링과-예외-문제-해결)
<!--te-->

JDBC 이해
=======
- `JDBC 등장 이유`
  - 각 데이터베이스 별로 커넥션을 연결하는 방법, SQL을 전달하는 방법, 그리고 결과를 응답 받는 방법이 모두 달라서 데이터베이스를 변경하면 데이터베이스 사용 코드도 함께 변경됐어야하며 개발자는 각각의 데이터베이스마다 커넥션 연결, SQL 전달, 그리고 그 결과를 응답 받는 방법을 학습해야했음
- `JDBC 표준 인터페이스`
  - java.sql.Connection - 연결
  - java.sql.Statement - SQL을 담은 내용
  - java.sql.ResultSet - SQL 요청 응답
  - 각각의 DB 벤더(회사)에서 자신의 DB에 맞도록 구현해서 JDBC 드라이버 라이브러리를 제공
- `JDBC와 최신 데이터 접근 기술 - SQL Mapper`
  - JDBC를 편리하게 사용하도록 도와주는 기술 중 하나
  - SQL 응답 결과를 객체로 편리하게 변환
  - JDBC의 반복 코드를 제거
  - SQL은 직접 작성해야함
  - 대표 기술: JdbcTemplate, MyBatis
- `JDBC와 최신 데이터 접근 기술 - ORM`
  - JDBC를 편리하게 사용하도록 도와주는 기술 중 하나
  - ORM은 객체를 관계형 데이터베이스 테이블과 매핑해주는 기술
  - 발자는 반복적인 SQL을 직접 작성하지 않고, ORM 기술이 개발자 대신에 SQL을 동적으로 만들어 실행
  - 대표 기술: JPA(인터페이스), 하이버네이트(JPA 구현체), 이클립스링크(JPA 구현체)


커넥션풀과 데이터소스 이해
=======
- `DB 커넥션 과정`
  - DB 드라이버는 DB와 TCP/IP 커넥션을 연결(3 way handshake)
  - DB 드라이버는 TCP/IP 커넥션이 연결되면 ID, PW와 기타 부가정보를 DB에 전달
  - DB는 ID, PW를 통해 내부 인증을 완료하고, 내부에 DB 세션을 생성
  - DB는 커넥션 생성이 완료되었다는 응답
  - DB 드라이버는 커넥션 객체를 생성해서 클라이언트에 반환
- `커넥션 풀`
  - 커넥션을 새로 만드는 것은 과정도 복잡하고 시간도 많이 많이 소모되어 커넥션을 미리 생성해두고 사용하는 방법
  - 애플리케이션 로직은 커넥션 풀을 통해 이미 생성되어 있는 커넥션을 객체 참조로 그냥 가져다 사용
  - 커넥션을 사용하고 나면 종료하는 것이 아니라 커넥션이 살아있는 상태로 커넥션 풀에 반환
  - 성능과 사용의 편리함 측면에서 최근에는 스프링에서 기본 제공하는 `hikariCP` 를 주로 사용
- `DataSource`
  - DataSource 는 커넥션을 획득하는 방법을 추상화 하는 인터페이스
  - JDBC DriverManager를 통해 신규 커넥션 획득 또는 커넥션 풀에서 조회하여 커넥션을 얻을 수 있는데 각 방법이 다르기 때문에 추상화하여 애플리케이션 로직은 변경되지 않도록 처리
  - `커넥션 풀`은 HikariCP 커넥션 풀의 코드를 직접 의존하는 것이 아니라 DataSource 인터페이스에만 의존
  - DriverManager는 DataSource 인터페이스를 사용하지 않지만 통일성 있게 DataSource 를 통해서 커넥션을 사용할 수 있도록 `DriverManagerDataSource` 라는 DataSource 를 구현한 클래스를 스프링에서 제공
- `DriverManager`
```java
@Slf4j
public class ConnectionTest {

    @Test
    void driverManager() throws SQLException {
      Connection con1 = DriverManager.getConnection(URL, USERNAME, PASSWORD); // DriverManager 는 커넥션을 획득할 때 마다 URL , USERNAME , PASSWORD 같은 파라미터를 계속 전달
      Connection con2 = DriverManager.getConnection(URL, USERNAME, PASSWORD); 
      log.info("connection={}, class={}", con1, con1.getClass()); 
      log.info("connection={}, class={}", con2, con2.getClass());
    }

    private void useDataSource(DataSource dataSource) throws SQLException { 
      Connection con1 = dataSource.getConnection(); // 사용 -> DataSource 를 사용하는 방식은 처음 객체를 생성할 때만 필요한 파리미터를 넘겨두고, 커넥션을 획득할 때는 단순히 dataSource.getConnection() 만 호출
      Connection con2 = dataSource.getConnection(); 
      log.info("connection={}, class={}", con1, con1.getClass()); 
      log.info("connection={}, class={}", con2, con2.getClass());
    }

    @Test
    void dataSourceDriverManager() throws SQLException {  
      //DriverManagerDataSource - 항상 새로운 커넥션 획득
      DriverManagerDataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD); // 설정
      useDataSource(dataSource);
    }

}
```
- `커넥션 풀`
```java
@Slf4j
public class ConnectionTest {

    @Test
    void dataSourceConnectionPool() throws SQLException, InterruptedException {
      //커넥션 풀링: HikariProxyConnection(Proxy) -> JdbcConnection(Target) 
      HikariDataSource dataSource = new HikariDataSource(); 
      dataSource.setJdbcUrl(URL);
      dataSource.setUsername(USERNAME); 
      dataSource.setPassword(PASSWORD); 
      dataSource.setMaximumPoolSize(10); // 커넥션 풀 최대 사이즈
      dataSource.setPoolName("MyPool"); // 풀의 이름
      
      useDataSource(dataSource);
    }

}
```

트랜잭션 이해
=======
- `트랜잭션`
  - 데이터베이스에 저장하는 가장 큰 이유는 트랜잭션을 지원하기 때문
  - `모든 작업이 성공해서 데이터베이스에 정상 반영하는 것은 커밋(Commit)`이고, 작업 중 `하나라도 실패해서 거래 이전으로 되돌리는 것은 롤백(Rollback)`
  - 자동 커밋: 기본 설정 모드로 자동 커밋은 각각의 쿼리를 트랜잭션 단위로 간주하여 쿼리 하나 실행 직후에 자동으로 커밋을 호출하여 커밋이나 롤백을 직접 호출하지 않아도 되는 편리함이 있지만 트랜잭션 기능을 수행할 수 없음
  - 수동 커밋: 트랜잭션을 시작할 때 수동 커밋 모드로 변경하고 실행
- `트랜잭션 ACID`
  - 원자성: 트랜잭션 내에서 실행한 작업들은 마치 하나의 작업인 것처럼 모두 성공 하거나 모두 실패해야함
  - 일관성: 모든 트랜잭션은 일관성 있는 데이터베이스 상태를 유지해야함 (무결성 제약 조건 등등..) 
  - 격리성: 동시에 실행되는 트랜잭션들이 서로에게 영향을 미치지 않도록 격리돼야함 (동시에 같은 데이터를 수정하지 못하도록 해야함)
  - 지속성: 트랜잭션을 성공적으로 끝내면 그 결과가 항상 기록돼야하며 문제가 발생되면 기록을 통해 복구가 가능해야함
- `트랜잭션 격리 수준`
  - 격리성은 동시성과 관련된 성능 이슈로 인해 트랜잭션 격리 수준 (Isolation level)을 선택할 수 있음
  - READ UNCOMMITED(커밋되지 않은 읽기) 
  - `READ COMMITTED(커밋된 읽기)` <- 주로 사용
  - REPEATABLE READ(반복 가능한 읽기)
  - SERIALIZABLE(직렬화 가능)
- `데이터베이스 연결 구조와 DB 세션`
  - 데이터 변경 쿼리를 실행하고 데이터베이스에 그 결과를 반영하려면 커밋 명령어인 `commit` 을 호출하고, 결과를 반영하고 싶지 않으면 롤백 명령어인 `rollback` 을 호출
  - 커밋을 호출하기 전까지는 임시로 데이터를 저장하는 것이므로 따라서 해당 트랜잭션을 시작한 세션(사용자)에게만 변경 데이터가 보이고 다른 세션(사용자)에게는 변경 데이터가 보이지 않음
- `DB 락`
  - 세션 하나가 트랜잭션을 시작하고 데이터를 수정하는 동안 아직 커밋을 수행하지 않았는데, 다른 세션에서 `동시에 같은 데이터를 수정`하게 되면 여러가지 문제가 발생하므로 세션이 트랜잭션을 시작하고 데이터를 수정하는 동안에는 커밋이나 롤백 전까지 다른 세션에서 해당 데이터를 수정할 수 없게 막아야함
  - 트랜잭션 수행 시에 락을 획득하는 매커니즘을 추가하여 다른 세션이 동일한 수정을 하지 못하도록 하고 트랜잭션이 끝났을 때 락을 반납
  - 보통 데이터를 조회할 때는 락을 획득하지 않고 바로 데이터를 조회하는데 데이터를 조회할 때도 다른 곳의 데이터의 수정을 막기 위해 락을 획득하고 싶을 때는 `select for update` 구문을 사용

스프링과 트랜잭션 문제 해결
=======
- `트랜잭션 추상화 및 동기화`
  - 트랜잭션을 추상화하지 않으면 트랜잭션 적용을 위해 `JDBC 및 JPA 구현 기술이 서비스 계층에 누수`되고 `예외(SQLException) 처리도 누수`되며 트랜잭션을 위한 `반복적인 JDBC 트랜잭션 코드 또는 JPA 트랜잭션 코드가 사용`되어 문제
  - 트랜잭션의 리소스를 동기화하지 않으면 커넥션이 달라져서 트랜잭션 처리에 문제가 발생하므로 파라미터로 커넥션을 넘겨줘야하는데 이는 중복 코드가 많이 생성됨
- `트랜잭션 매니저와 트랜잭션 동기화 매니저`
  - 트랜잭션 매니저: JDBC 및 JPA 구현 기술 등등의 다양한 구현 기술을 통일한 인터페이스
  ```java
  public interface PlatformTransactionManager extends TransactionManager {
    TransactionStatus getTransaction(@Nullable TransactionDefinition definition) throws TransactionException; //  트랜잭션을 시작
    void commit(TransactionStatus status) throws TransactionException; // 트랜잭션을 커밋
    void rollback(TransactionStatus status) throws TransactionException; // 트랜잭션을 롤백
  }
  ```
    - JdbcTransactionManager: JDBC 트랜잭션 기능을 제공하는 PlatformTransactionManager의 구현체
    - JpaTransactionManager: JPA 트랜잭션 기능을 제공하는 PlatformTransactionManager의 구현체
  - 트랜잭션 동기화 매니저: 쓰레드 로컬을 사용해서 멀티쓰레드 상황에서도 안전하게 커넥션을 동기화하여 커넥션을 보관하고 관리해줌
- `트랜잭션 템플릿`
  - 트랜잭션을 시작하고 비즈니스 로직을 실행하고 성공하면 커밋하고 예외가 발생해서 실패하면 롤백하는 과정이 반복되는데 반복하는 코드를 많이 제거하기 위해 트랜잭션 템플릿 사용
  ```java
  public class TransactionTemplate {
      private PlatformTransactionManager transactionManager;
      public <T> T execute(TransactionCallback<T> action){..} // 응답 값이 있을 때 사용
      void executeWithoutResult(Consumer<TransactionStatus> action){..} // 응답 값이 없을 때 사용
  }
  ```
- `트랙잭션 AOP`
  - 트랜잭션 매니저 및 트랜잭션 템플릿을 통합하여 프록시 기반와 스프링 AOP를 기반으로 제공하는 `@Transactional`을 사용하면 스프링이 트랜잭션을 편리하게 처리
  - @Transactional 애노테이션은 메서드에 붙이면 해당 메서드만 AOP의 대상이 되고 클래스에 붙이면 외부에서 호출 가능한 public 메서드가 AOP 적용 대상이 됨
  - 어드바이저: BeanFactoryTransactionAttributeSourceAdvisor
  - 포인트컷: TransactionAttributeSourcePointcut
  - 어드바이스: TransactionInterceptor
  ```java
  @Slf4j
  @RequiredArgsConstructor
  public class MemberServiceV3_3 {

    private final MemberRepositoryV3 memberRepository;
    
    @Transactional
    public void accountTransfer(String fromId, String toId, int money) throws SQLException {
        bizLogic(fromId, toId, money);
    }

    private void bizLogic(String fromId, String toId, int money) throws SQLException {
      Member fromMember = memberRepository.findById(fromId); 
      Member toMember = memberRepository.findById(toId);
      memberRepository.update(fromId, fromMember.getMoney() - money); 
      validation(toMember);
      memberRepository.update(toId, toMember.getMoney() + money);
    }
    
    private void validation(Member toMember) {
      if (toMember.getMemberId().equals("ex")) {
        throw new IllegalStateException("이체중 예외 발생"); 
      }
    }   
  }
  ```
- `스프링 부트의 자동 리소스 등록`
  - 스프링 부트가 등장하기 이전에는 데이터 소스와 트랜잭션 매니저를 개발자가 직접 스프링 빈으로 등록해서 사용
  - 데이터 소스는 application.properties 에 있는 속성을 사용해서 DataSource(HikariDataSource)를 생성
  - 스프링 부트는 적절한 트랜잭션 매니저(PlatformTransactionManager)를 자동으로 스프링 빈에 등록

자바 예외 이해
=======

스프링과 예외 문제 해결
=======
