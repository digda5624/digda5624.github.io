---
title: "트랜잭션 알기"
categories:
- Spring 
toc: true
toc_sticky: true
toc_label: "트랜잭션"
tags:
- [spring, jpa, database]
---

# 🚀 트랜잭션

## 🤔 트랜잭션 알기
데이터베이스를 사용하는 이유?
1. transaction (commit 과 rollback)
2. 다른 서비스와의 데이터 공유
3. 동시성

### 트랜잭션의 ACID
1. 원자성<br>
   트랜잭션 내에서 실행한 작업들은 마치 하나의 작업인 것처럼 모두 성공하거나 모두 실패해야 한다.
2. 일관성<br>
   모든 트랜잭션은 일관성 있는 데이터베이스 상태를 유지해야 한다.
3. 격리성<br>
   동시에 실행되는 트랜잭션들이 서로에게 영향을 미치지 않도록 격리한다.<br>
   transaction의 isolation level을 생각하면 된다.<br>

> ### 트랜잭션 격리 수준 (Isolation level)
> - READ UNCOMMITED(커밋되지 않은 읽기)
> - READ COMMITTED(커밋된 읽기)
> - REPEATABLE READ(반복 가능한 읽기)
> - SERIALIZABLE(직렬화 가능)

참고로 JPA에서는 데이터베이스에 관계없이 1차캐시(영속성 컨텍스트)를 이용, Repeatable Read를 지원한다.
4. 지속성<br>
   트랜잭션을 성공적으로 끝내면 그 결과가 항상 기록되어야 한다. 중각에 시스템에 문제가 발생해도 데이터베이스 로그 등을
   사용 해서 성공한 트랜잭션 내용을 복구해야한다.

### 들어가기전...
DataBase의 세션 개념을 알아갈 필요가 있다.
1. 커넥션을 맺고 난뒤 해당 커넥션에 맞게 데이터베이스 서버는 세션을 생성한다.
2. 커넥션을 통한 모든 요청은 각 세션을 통해서 실행된다.
3. 세션은 트랜잭션을 시작하고 커밋 또는 롤백을 통해 트랜잭션을 종료한다.
4. 여기서 세션별로 다시 테이블을 가진다고 생각하면 편하다. (트랜잭션 격리를 위함)<br>
   쉽게 말하자면 2개의 세션이 있다고 가정했을 때 한쪽 세션에서 트랜잭션 시작후 insert, update, delete를 한다고 해서
   다른 세션에서 그 결과를 볼 수 있는 것이 아니다. commit을 해야 보인다.

![img](https://user-images.githubusercontent.com/69373314/190559348-1ef9b990-29e6-4d7f-b232-f1f62b2e63cf.png)

### 🤣 락의 필요성 등장
기본적으로 대부분의 데이터베이스들은 select 시점에 commit 된 결과를 가져오는 READ COMMITTED 이상의 트랜잭션 레벨과
REPEATABLE READ 이하의 트랜잭션 레벨을 제공한다.<br>
이 이유는 어플리케이션에게 최대한 동시성을 제공하려는 db의 노력을 볼 수 있다.<br>
직렬화 레벨의 경우 단순 조회 쿼리에도 다른 트랜잭션에서 해당 row를 조회하고 있다면 대기가 걸리기 때문에 성능 저하의 이슈가 있기 때문이다.
곧, 그 말은 대부분의 db들은 select 레벨에서 조회시에 대기(락)를 하지 않는다.

고로 다음과 같은 문제가 발생 할 수 있다.(for update로 락을 걸지 않고 가져오는 경우, JPA dirty check로 설명)
1. 2개의 트랜잭션이 존재한다.
2. 1번 트랜잭션에서 먼저 계좌를 가져온다.
3. 2번 트랜잭션에서 또 계좌 정보를 가져온다.
4. 1번 트랜잭션에서 계좌에서-2000 원을 한다
5. 1번 트랜잭션에서 commit
6. 2번 트랜잭션에서 계좌에서 +2000 원을 한다.
7. 2번 트랜잭션 commit
8. 결과적으로 계좌는 +2000원이 되었다.(덮어씌워짐)

락을 걸지 않을 경우 다음과 같은 문제가 발생하게 된다.<br>
고로 어플리케이션 레벨에서 sql에 변경을 위한 조회일 경우 for update를 통해(조회 락) 데이터를 조회하는 것이 필요하다.<br>
=> 주의 할점은 for update를 통해 가져와도 다른 트랜잭션에서 select로 가져오면 말짱 도로묵이다.

후에 락에 대해서 좀더 포스팅 해봐야겠다.

### application + transaction
트랜잭션 도중 어플리케이션에서 오류가 발생할 경우 try-catch를 이용해 데이터베이스를 rollback 시킬 수 있다.<br>
그렇다면 어플리케이션에서 트랜잭션을 어디에서 시작하고 끝내야 할까<br>
1. 어플리케이션에서 오류가 나는 곳은 비지니스 로직(서비스 계층) 부분이다.
2. 그말은 곧, 하나의 request는 같은 트랜잭션을 사용 즉 같은 커넥션을 유지해야 한다.

트랜잭션을 사용하여 Jdbc를 사용해보자

## jdbc + transaction

🖥 Repository
```java
@Slf4j
@RequiredArgsConstructor
public class MemberRepository {

    private final DataSource dataSource;

    public Member save(Member member) throws SQLException {
        String sql = "insert into member(member_id, money) values (?, ?)";

        Connection con = null;
        PreparedStatement pstmt = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, member.getMemberId());
            pstmt.setInt(2, member.getMoney());
            pstmt.executeUpdate();
            return member;
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }

    }

    public Member findById(String memberId) throws SQLException {
        String sql = "select * from member where member_id = ?";

        Connection con = null;
        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, memberId);

            rs = pstmt.executeQuery();
            if (rs.next()) {
                Member member = new Member();
                member.setMemberId(rs.getString("member_id"));
                member.setMoney(rs.getInt("money"));
                return member;
            } else {
                throw new NoSuchElementException("member not found memberId=" + memberId);
            }

        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, rs);
        }

    }

    public Member findById(Connection con, String memberId) throws SQLException {
        String sql = "select * from member where member_id = ?";

        PreparedStatement pstmt = null;
        ResultSet rs = null;

        try {
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, memberId);

            rs = pstmt.executeQuery();
            if (rs.next()) {
                Member member = new Member();
                member.setMemberId(rs.getString("member_id"));
                member.setMoney(rs.getInt("money"));
                return member;
            } else {
                throw new NoSuchElementException("member not found memberId=" + memberId);
            }

        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            //connection은 여기서 닫지 않는다.
            JdbcUtils.closeResultSet(rs);
            JdbcUtils.closeStatement(pstmt);
        }

    }

    public void update(String memberId, int money) throws SQLException {
        String sql = "update member set money=? where member_id=?";

        Connection con = null;
        PreparedStatement pstmt = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setInt(1, money);
            pstmt.setString(2, memberId);
            int resultSize = pstmt.executeUpdate();
            log.info("resultSize={}", resultSize);
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }

    }

    public void update(Connection con, String memberId, int money) throws SQLException {
        String sql = "update member set money=? where member_id=?";

        PreparedStatement pstmt = null;

        try {
            pstmt = con.prepareStatement(sql);
            pstmt.setInt(1, money);
            pstmt.setString(2, memberId);
            int resultSize = pstmt.executeUpdate();
            log.info("resultSize={}", resultSize);
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            //connection은 여기서 닫지 않는다.
            JdbcUtils.closeStatement(pstmt);
        }

    }

    public void delete(String memberId) throws SQLException {
        String sql = "delete from member where member_id=?";

        Connection con = null;
        PreparedStatement pstmt = null;

        try {
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, memberId);
            pstmt.executeUpdate();
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }

    }

    private void close(Connection con, Statement stmt, ResultSet rs) {
        JdbcUtils.closeResultSet(rs);
        JdbcUtils.closeStatement(stmt);
        JdbcUtils.closeConnection(con);
    }
    
    private Connection getConnection() throws SQLException {
        Connection con = dataSource.getConnection();
        log.info("get connection={}, class={}", con, con.getClass());
        return con;
    }
}
```

🖥 Service
```java
@Slf4j
@RequiredArgsConstructor
public class MemberService {

    private final DataSource dataSource;
    private final MemberRepository memberRepository;

    public void accountTransfer(String fromId, String toId, int money) throws SQLException {
        Connection con = dataSource.getConnection();
        try {
            con.setAutoCommit(false);//트랜잭션 시작
            //비즈니스 로직
            bizLogic(con, fromId, toId, money);
            con.commit(); //성공시 커밋
        } catch (Exception e) {
            con.rollback(); //실패시 롤백
            throw new IllegalStateException(e);
        } finally {
            release(con);
        }

    }

    private void bizLogic(Connection con, String fromId, String toId, int money) throws SQLException {
        Member fromMember = memberRepository.findById(con, fromId);
        Member toMember = memberRepository.findById(con, toId);

        memberRepository.update(con, fromId, fromMember.getMoney() - money);
        validation(toMember);
        memberRepository.update(con, toId, toMember.getMoney() + money);
    }

    private void validation(Member toMember) {
        if (toMember.getMemberId().equals("ex")) {
            throw new IllegalStateException("이체중 예외 발생");
        }
    }

    private void release(Connection con) {
        if (con != null) {
            try {
                con.setAutoCommit(true); //커넥션 풀 고려
                con.close();
            } catch (Exception e) {
                log.info("error", e);
            }
        }
    }
}
```
1. 같은 트랜잭션 유지를 유해 같은 커넥션을 사용해야한다.
2. 서비스 계층에서 커넥션을 만들고, 끝날때 트랜잭션을 커밋하고 커넥션을 종료해야한다.

![img_1](https://user-images.githubusercontent.com/69373314/190559363-4ed732fe-5c58-40fd-842a-eb422575b964.png)

핵심들은 같은 커넥션을 유지하기 위해서 service에서 repository 쪽으로 connection을 전달해야하며
service 계층에서 커넥션을 얻고 commit 혹은 rollback 하는 과정이 매우 겹친다고 볼 수 있다.

애플리케이션에서 DB 트랜잭션을 적용하려면 서비스 계층이 매우 지저분해지고 커넥션을 repo 레벨로 계속 던져 주어
야 하기 때문에(같은 트랜잭션을 사용하려면) 유지 보수에도 쉽지 않아 진다.

## 문제점
1. 서비스 계층은 특정 기술에 의존하지 않고 순수 java 코드로 작성하는 편이 좋다<br>
   현재 서비스 계층은 jdbc를 의존하고 있다.
   * SQLException
   * DataSource
   * Connection


2. 트랜잭션을 사용하기 위해서 jdbc 기술에 의존 하게 된다. jdbc는 데이터 접근 기술 중 하나임을
   잊지 말자(물론 모든 db 접근 기술들은 jdbc를 내부적으로 사용하고 있다.) jdbc 에서는 connection을 통해
   setAutoCommit으로 트랜잭션을 열고 commit, rollback 으로 트랜잭션을 닫는다.


3. 2번의 이유로 서비스에서 특정 계층을 의존하고 있으므로 JDBC에서 JPA 같은 기술로 바꾸어
   사용하게 되면 서비스 코드 또한 고쳐져야한다.

### 문제점 정리
1. jdbc 구현 기술이 서비스 계층에 누수 되었다.
   * 트랜잭션을 적용하기 위해 jdbc 구현 기술이 서비스 계층에 누수
   * 서비스 계층은 순수 java 코드로 만들고 특정 기술에 의존하는 행위들은 모두 spring에서는
     component 로 관리하거나 따로 Class로 관리하는 것이 좋다.
      * 물론 따로 빼놔도 Transaction을 적용하게 될 경우 같은 Connection 즉, 같은 세션을
        유지하기 위해서 jdbc connection을 사용하면서 service가 특정 계층을 의존하게 되었다.


2. 트랜잭션 동기화 문제
   * 단순 조회의 경우 트랜잭션을 유지할 필요가 없다.


3. 트랜잭션을 위한 반복 코드
   * try - catch - finally 같은 리소스 할당 해제 같은 반복적인 코드가 반복된다.


## 🤔 데이터 접근 기술 추상화
특정기술에 의존하지 않기 위해서 결국 interface를 통한 추상화가 필요하다.<br>
이전에 배웠던, 커넥션 풀과 DriverManger를 추상하기 위한 DataSource interface나
다양한 db Driver를 관리하는 DriverManger 들이 그러하다.

데이터 접근 기술에 따라서 트랜잭션을 시작하는 방식이 모두 다르다.<br>
jdbc는 connection.setAutoCommit 을 통하여<br>
jpa 는 transaction.begin 을 통하여<br>
mybatis는 sqlMapClient.startTransaction 을 통하여 진행한다.<br>

고로 이런 데이터 접근 계층을 Class 하나씩으로 구현하게 될경우 특정 기술에 의존적이다
따라서 이전에도 해봤듯이 각각의 Class들을 하나로 묶어줄 interface가 필요하게 된다.

service 계층에서 사용할 repository 레벨을 interface로 정의 한뒤 각 구현체들을 상황에 맞게
끼워 맞추면 되는 것이다. 구현체가 변경이 된다고 해서 service 의 계층이 변화되지 않게 바뀐다.

## spring + 추상화
하지만 위처럼 추상화를 한다고 해서 반복적인 코드가 사라지는 것은 아니다. service계층에서
repo 계층의 의존성을 줄인 것이지 interface 를 구현하고 있는 부분은 반복적인 코드가 겹쳐지게 된다.
따라서 Transaction 자체를 추상화할 필요가 있으며 jpa, jdbc, mybatis 같은 다양한
데이터 접근 기술들에 대한 transaction 관리를 하나의 인터페이스로 적용 시켜야 한다.

![img_2](https://user-images.githubusercontent.com/69373314/190559469-0623a1e3-7051-4055-b445-74fd5cba8e27.png)

spring은 각각 데이터 접근 기술에 대해 TransactionManager를 구상했고 스프링에서
PlatformTransactionManager interface를 통해 각각의 데이터 접근 기술들의 transaction
을 관리한다.

🖥 PlatformTransactionManager
```java
public interface PlatformTransactionManager extends TransactionManager {
    
      TransactionStatus getTransaction(@Nullable TransactionDefinition definition)  throws TransactionException;
      
      void commit(TransactionStatus status) throws TransactionException;
      
      void rollback(TransactionStatus status) throws TransactionException;
    
    }
```

또한 예외도 TransactionException으로 한번 더 감싸 Exception을 추상화 한것을 볼수 있다.
getTransaction은 이미 시작한 Transaction이 존재할 경우 같은 트랜잭션을 유지 시켜준다.

스프링이 제공하는 interface은 크게 2가지 역할을 한다.
* 트랜잭션 추상화
* 리소스 동기화 (같은 트랜잭션 유지)<br>
  같은 트랜잭션을 유지하기 위해 (즉, 같은 커넥션을 사용하기 위해) 파라미터로 커넥션을 전달 했는데 이제 로컬 스레드를
  사용하여 파라미터로 넘기지 않아도 된다.<br>

🤔 로컬 스레드 - 스레드 별로 임시저장소를 가진다고 생각하면 편하다. 단, thread pool 을 사용하는 상황일경우
주의해서 다뤄야한다.

스프링은 트랜잭션 동기화를 위한 별도의 클래스를 제공한다. (같은 커넥션을 꺼내기 위해서)<br>
org.springframework.transaction.support.TransactionSynchronizationManager

![img_3](https://user-images.githubusercontent.com/69373314/190559477-19624fe9-6a12-4850-9b27-bbbedae16209.png)

동작 순서는 다음과 같다
1. 트랜잭션 매니저는 DataSource를 통해 커넥션을 만들고 트랜잭션을 시작한다.
2. 트랜잭션 매니저는 트랜잭션이 시작된 커넥션을 트랜잭션 동기화 매니저에 보관
3. repo 계층은 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼내서 사용한다.
4. 트랜잭션이 종료되면 트랜잭션 매니저는 트랜잭션 동기화 매니저에 보관된 커넥션을 통해 트랜잭션을 종료, 리소르를 해제한다.

### 클래스 정리
DataSource - jdbc DriverManager, Thead Pool Connection 추상화<br>
PlatformTransactionManger - 각기 다른 데이터 접근 기술의 Transaction 관리, 동기화 추상화<br>
TransactionSynchronizationManager - 같은 커넥션 혹은 자원(EntityManager) 을 사용하게 해주는 support
글래스

### 현재까지의 문제점 정리
목표 : Transaction 과 비지니스 로직의 분리
1. 현재 jdbc 기술에 의존하고 있다. <br>
   => 다른 db 접근 기술로 바뀌게 될경우 service 계층을 바꾸어야 한다.


2. db접근 기술들을 통합하려고 interface를 사용해도 service 단에서는 계속해서 Transaction 시작과 종료 코드가 드러난다.<br>
   추가적으로 여전히 dataSource를 repository 계층으로 파라미터로 넘겨서 같은 Connection 을 유지해야 한다.<br>
   (같은 Transaction 유지를 위함)


3. 2번을 해결하기 위해서 repository에서 같은 Connection 사용을 위해 동기화 TransactionManager 적용이 필요하다.<br>
   서비스 계층에서는 PlatformTransactionManager를 사용하여 디비 접근 기술에 대한 의존성을 줄인다.


4. service 계층에서 반복적인 try - catch 및 반복성 있는 코드

#### 해결을 위한 코드
🖥 TestCode 에서 의존성 주입 단계
```java
/**
 * 트랜잭션 - 트랜잭션 매니저
 */
class MemberServiceTest {
    
    private MemberRepository memberRepository;
    private MemberService memberService;

    // 의존성 주입 단계
    @BeforeEach
    void before() {
        // DataSource 를 사용하여 어떤 DB 사용할것인지 + 커넥션 풀 사용 or jdbc Connection 사용할 것인지 정함
        // 현재는 jdbc Connection 지원하는 spring DriverManagerDataSource 사용
        DataSource dataSource = new DriverManagerDataSource(URL, USERNAME, PASSWORD);
        memberRepository = new MemberRepositoryV3(dataSource);
        // 어떤 디비접근 기술을 사용할 것인지를 명시 jdbc를 사용할 것이므로 DataSourceTrnasactionManager 주입
        // JPA 사용 할 경우 JpaTransactionManager 로 주입
        // 결국 바뀌는 것은 Config 파일 즉, 의존성을 주입시켜주는 코드만 바뀔 뿐이다.
        PlatformTransactionManager transactionManager = new DataSourceTransactionManager(dataSource);
        memberService = new MemberServiceV3_1(transactionManager, memberRepository);
    }
    
    // ... test 코드들 작성
}
```

🖥 service 계층
```java
@Slf4j
@RequiredArgsConstructor
public class MemberService {

    private final PlatformTransactionManager transactionManager;
    private final MemberRepository memberRepository;

    public void accountTransfer(String fromId, String toId, int money) throws SQLException {
        //트랜잭션 시작
        //Transaction 에 대한 설정을 작성하기 위해 DefaultTransactionDefinition 을 주입한다,
        TransactionStatus status = transactionManager.getTransaction(new DefaultTransactionDefinition());

        try {
            //비즈니스 로직
            bizLogic(fromId, toId, money);
            transactionManager.commit(status); //성공시 커밋
        } catch (Exception e) {
            transactionManager.rollback(status); //실패시 롤백
            throw new IllegalStateException(e);
        }

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

🖥 repository 계층
```java
/**
 * 트랜잭션 - 트랜잭션 매니저
 * DataSourceUtils.getConnection()
 * DataSourceUtils.releaseConnection()
 */
@Slf4j
public class MemberRepository {

    private final DataSource dataSource;

    public MemberRepository(DataSource dataSource) {
        this.dataSource = dataSource;
    }

    public Member save(Member member) throws SQLException {
        String sql = "insert into member(member_id, money) values (?, ?)";

        Connection con = null;
        PreparedStatement pstmt = null;

        try {
            // 같은 connection 사용을 위해 DataSourceUtils 를 사용한다. 
            // 내부적으로 Transaction 동기화 매니저를 사용하여 같은 Connection을 보장한다( Local Thread 사용 )
            con = getConnection();
            pstmt = con.prepareStatement(sql);
            pstmt.setString(1, member.getMemberId());
            pstmt.setInt(2, member.getMoney());
            pstmt.executeUpdate();
            return member;
        } catch (SQLException e) {
            log.error("db error", e);
            throw e;
        } finally {
            close(con, pstmt, null);
        }
    }
    
    private void close(Connection con, Statement stmt, ResultSet rs) {
        JdbcUtils.closeResultSet(rs);
        JdbcUtils.closeStatement(stmt);
        //주의! 트랜잭션 동기화를 사용하려면 DataSourceUtils를 사용해야 한다.
        DataSourceUtils.releaseConnection(con, dataSource);
    }


    private Connection getConnection() throws SQLException {
        //주의! 트랜잭션 동기화를 사용하려면 DataSourceUtils를 사용해야 한다.
        Connection con = DataSourceUtils.getConnection(dataSource);
        log.info("get connection={}, class={}", con, con.getClass());
        return con;
    }
```

#### 법칙
1. DB 에 대한 정보는 DataSource 를 통해 얻어야한다. (항상 jdbc 를 사용하게 된다. Connection Pool 또한)
2. service 를 시작할 때 Transaction 을 열고 닫는 부분이 있어야한다.
3. repository 는 같은 Service 메서드에 묶여있을 때 항상 같은 Connection 을 공유해야 한다.

* 결국에 큰 틀은 변하지 않는다. 어떤 데이터 접근 기술을 써도 Connection 을 생성하기 위해 DataSource 가 필요하다.<br>
  DataSource는 DB 설정파일을 통해 관리한다. (물론 어떤 Connection 을 의존하느냐에 따라 추가적인 세팅은 필요할 것이다)<br>


* Service 계층에서 트랜잭션을 시작하기 위해 Connection을 얻고 트랜잭션 매니저를 통해서 트랜잭션을 시작한다.<br>
  (Connection을 얻을때 DataSource를 통해 얻는다)<br>


* Repository 계층에서 같은 Connection을 사용하기 위해서 트랜잭션 동기화 매니저를 사용하며 마찬가지로 Connection을
  얻는 과정이기 때문에 dataSource를 필요로 한다.


### 흐름 정리
1. 서비스 계층에서 TransactionManager.getTransaction() 으로 트랜잭션 시작
2. 트랜잭션을 시작하기위해 TransactionManager 내부에 초기화된 DataSource로 커넥션을 얻는다.
3. 얻은 커넥션을 setAutocommit(false) 를 통해 실제 데이터 베이스 트랜잭션을 시작한다. (jdbc 기반)
4. 커넥션을 Transaction 동기화 Manager에 보관(Local Thread 사용)
5. 서비스는 비즈니스 로직을 실행하면서 리포지토리의 메서드들을 호출한다. 동기화 매니저를 통해 같은 connection을 사용
6. 위에 코드에서는 DataSourceUtils.getConnection()을 사용하여 트랜잭션 동기화 매니저에 보관된 커넥션을 꺼낸다.
7. 각 repository 에 SQL 전달하여 소통
8. 트랜잭션을 종료한다. commit or rollback
9. 리소스를 정리한다
   * Thead Local 정리
   * setAutoCommit 정리
   * 커넥션 종료 or 반납

![img_4](https://user-images.githubusercontent.com/69373314/190559540-3f6a4fb7-24be-4d32-a744-7265a77cf657.png)
![img_5](https://user-images.githubusercontent.com/69373314/190559546-14a5332c-a774-4d51-8eff-3dd74d8cddd9.png)
![img_6](https://user-images.githubusercontent.com/69373314/190559551-5fbc4cd8-3092-4f3c-98ec-94af5e32c076.png)

#### ✅ 현재까지의 해결
1. 트랜잭션 추상화(interface + Local Thread) 를 사용하여 서비스 계층에서 JDBC 기술에 의존하지 않는다.
2. 더이상 커넥션을 파라미터로 넘기지 않아도 된다.

=> but, 반복적인 코드는 아직 유효하다.

### 반복적인 코드 고치기
현재는 try - catch - finally 와 트랜잭션을 시작 - 종료상황이 반복적으로 쓰여지고 있다.<br>

따라서 템플릿 콜백 패턴을 활용해서 반복문제를 해결 해야 한다.

## 😼 template - callback 패턴?

> ### 콜백 이란?<br>
> 프로그래밍에서 콜백은 다른 코드의 인수(파라미터)로서 넘겨주는 실행 가능한 코드를 말한다. 콜백을 넘겨받는 코드는
> 필요에 따라 즉시 실행할 수도 있고, 나중에 실행 할 수도있다.

java 에서는 코드를 인수로 넘기려면 객체화를 시켜야 할텐데 java 8 부터는 람다식이 있다.<br>
원래는 하나의 메소드를 가진 인터페이스 구현하고 익명 내부 클래스 사용해서 만들었다.<br>
인터페이스 구성할 때 인자 정해주고, 반환타입 정해준다.

코드로 살피면 이런 흐름이다.<br>
🖥 template - callback
```java
@Slf4j
public class CallbackTest {

    public interface Callback<T>{
        T call();
    }

    public static class Template {
        public <T> T execute(Callback<T> callback){
            log.info("do something");
            T call = callback.call();
            log.info("end");
            return call;
        }
    }

    @Test
    void testCallback(){
        Template template = new Template();
        String result = template.execute(() -> {
            log.info("callback execute");
            return "test";
        });
        Assertions.assertThat(result).isEqualTo("test");
    }

    @Test
    void testCallback2(){
        Template template = new Template();
        String result = template.<String>execute(() -> {
            log.info("callback execute");
            return "test";
        });
        Assertions.assertThat(result).isEqualTo("test");
    }
}
```

spring 은 이를 위해 TransactionTemplate 을 제공하며 callback은 비지니스 로직으로 치환된다.<br>
service 를 수정하면 다음과 같아진다.

🖥 TransactionTemplate + bizlogic(callback)
```java
@Slf4j
public class MemberServiceV3_2 {

    private final TransactionTemplate txTemplate;
    private final MemberRepositoryV3 memberRepository;

    public MemberServiceV3_2(PlatformTransactionManager transactionManager, MemberRepositoryV3 memberRepository) {
        this.txTemplate = new TransactionTemplate(transactionManager);
        this.memberRepository = memberRepository;
    }

    public void accountTransfer(String fromId, String toId, int money) throws SQLException {
        txTemplate.executeWithoutResult((status) -> {
            //비즈니스 로직
            try {
                bizLogic(fromId, toId, money);
            } catch (SQLException e) {
                // 아쉽게도 람다식에서는 checked Exception을 무조건 처리해야 해서 try-catch가 감싸진다.
                throw new IllegalStateException(e);
            }
        });
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

### ⚡ 여전한 문제
반복적인 코드를 줄였음에도 람다식을 사용했기 때문에 Exception 처리를 해줘야하고 서비스 로직에서 Transaction을 호출하는
코드는 계속 남는다. 결국 DI의 힘을 빌려한다. (proxy 기술 사용)

동적프록시를 사용하여 코드를 단순화 해야한다.

프록시 + AOP를 적용하여 문제를 해결 할 수 있다.<br>
=> 해당 내용을 @Transacation 으로 해결하는 것이다.<br>
프록시를 통한 구현은 더 공부가 필요하다...<br>
spring 에서는 동적 프록시를 사용하나 일반적인 프록시 개념으로 적용을 한다면 다음과 같은 모습이다.

🖥 proxy 적용
```java
public class TransactionProxy {
    
    private MemberService target;
    
    public void logic() { //트랜잭션 시작
        TransactionStatus status = transactionManager.getTransaction(..);
        try {
            //실제 대상 호출 target.logic();
            transactionManager.commit(status); //성공시 커밋 
        } catch (Exception e) {
            transactionManager.rollback(status); //실패시 롤백
            throw new IllegalStateException(e);
        }
    } 
}
```

🖥 proxy 실제 구현 해본것 추가
```java
@AllArgsConstructor
@Slf4j
public class ProxyHandler implements InvocationHandler {

    private Object target;
    // 트랜잭션 기능에 필요한 것 추상화 해놓은 상황
    private PlatformTransactionManager mainTxManager;
    private PlatformTransactionManager secondTxManager;

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {

        PlatformTransactionManager txManager = getCurTxManger(method);
        TransactionStatus transactionStatus = txManager.getTransaction(new DefaultTransactionDefinition());

        try {
            Object result = method.invoke(target, args);
            txManager.commit(transactionStatus);
            return result;
        } catch (Exception exception) {
            txManager.rollback(transactionStatus);
            throw exception;
        }
    }

    private PlatformTransactionManager getCurTxManger(Method method){
        if(method.getName().startsWith("main")){
            log.info("main TxManager 호출 {}", mainTxManager.getClass());
            return mainTxManager;
        } else {
            log.info("second TxManager 호출 {}", secondTxManager.getClass());
            return secondTxManager;
        }
    }
}
```
대략 프록시는 대상(target) 의 기능외에 추가 할때 사용을 많이 하는데 마침 전후로 Transaction을
처리해야 하므로 프록시를 사용하기가 적절하다.<br>

결국 @Transaction 어노테이션 하나만으로 관심사가 최종적으로 분리되었으며 service 계층은 db 접근 기술을
의존하지 않게 변했다.

추가적으로 spring @Transaction 의 AOP 를 적용하기 위해서 다음 클래스들이 bean 으로 등록된다.

> 어드바이저: BeanFactoryTransactionAttributeSourceAdvisor<br>
> 포인트컷: TransactionAttributeSourcePointcut<br>
> 어드바이스: TransactionInterceptor

![img_7](https://user-images.githubusercontent.com/69373314/190559559-f76a9cdc-aa81-4eb7-b00d-6624a6c77a1d.png)

## ✅ 리소스 관리하기
결국 어플리케이션 레벨에서 관리해야 할 클래스는 크게 2가지 이다.
1. dataSource
2. TransactionManager
3. TransactionSynchronizationManager => 하지만 개발자가 관리할 일은 없다 TransactionManager 가 동기화를 위해
   알아서 사용한다.

스프링 부트는 autoConfiguration 으로 데이터소스와 라이브러리에 등록된 데이터 접근 기술을 사용하여 TransactionManager를
자동으로 등록한다.

스프링을 사용한다면 직접 dataSource와 내가 사용할 기술에 대한 TransactionManager를 등록해야한다.

🖥 직접 빈 등록
```java
public class Configure {
   @Bean
   DataSource dataSource() {
      // 해당 dataSource 정보는 설정파일로 관리하여 env 로 뽑아내던, static 변수로 관리를 하던 한다.
      return new DriverManagerDataSource(URL, USERNAME, PASSWORD);
   }

   @Bean
   PlatformTransactionManager transactionManager() {
      return new DataSourceTransactionManager(dataSource());
   }
}
```

자동 등록은 그저 application.yml 혹은 application.properties 로 관리를 한다.

스프링부트가 기본으로 생성하는 데이터소스는 커넥션풀을 제공하는 HikariDataSource 이다.
spring.datasource.url 속성이 없다면 내장 데이터베이스(메모리 DB)를 생성하려고 시도한다.

> #### application.properties
> spring.datasource.url=jdbc:h2:tcp://localhost/~/test<br>
> spring.datasource.username=username<br>
> spring.datasource.password=

