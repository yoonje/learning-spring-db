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

스프링과 트랜잭션 문제 해결
=======

자바 예외 이해
=======

스프링과 예외 문제 해결
=======
