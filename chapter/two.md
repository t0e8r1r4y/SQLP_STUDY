# 트랜잭션과 Lock

## 1. 트랜잭션 동시성 제어

### 1-1. 동시성 제어

- 동시성 제어란 동시에 실행되는 트랜잭션 수를 최대화하면서도 입력, 수정, 삭제, 검색 시 데이터의 무결성이 유지될 수 있도록 노력하는 것
- 서로 간섭을 일으키는 현상을 최소화하면서 데이터의 일관성과 무결성이 보장되도록 개발되어야 함

### 1-2. 트랜잭션이란?

- database transaction is a sequence of multiple operations performed on a database, and all served as a single logical unit of work

### 1-3. 트랜잭션의 특징(ACID)

- [서적보다 영문 내용이 더 정확한거 같아서 하나 정리한 링크](https://github.com/t0e8r1r4y/blogContents/blob/main/DEV/Database/MoreAboutDatabase/transaction%26acid.md)
- 원자성 : 모든 작업이 반영되거나 롤백된다. → 수행중인 트랜잭션에 의해 변경된 내용은 `롤백 세그번트` 에 저장. 오류가 발생하는 경우 `롤백 세그먼트`에 저장된 정보로 롤백하여 원자성 보장
- 일관성 : 트랜잭션 수행 전, 후에 데이터 모델의 모든 제약 조건을 만족하는 것을 보장
- 격리성 : 트랜잭션은 격리된다. DB마다 그 수준은 다르다.
- 영속성 : 한번 커밋 된 내용은 영구 적용 됨.

---

## 2. 트랜잭션 수준 읽기 일관성

### 2-1. 트랜잭션 수준 읽기 일관성이란?

- 오라클은 완벽한 문장 수준의 읽기 일관성을 보장하지만, 트랜잭션에 대해서는 기본적으로 보장하지 않는다.

### 2-2. 트랜잭션 고립화 수준

- ANSI/ISO SQL standard(SQL92)에서 정의하고 있는 아래 네가지 트랜잭션 고립화 수준을 정리함
- 레벨 0 = `Read Uncommitted`
    - 트랜잭션에 처리 중인, 아직 커멧되지 않은 데이터를 다른 트랜잭션이 읽는 것을 허용
    - Dirty Read, Non-Repeatable Read, Phantom Read 현상 발생
- 레벨 1 = `Read Committed`
    - Dirty Read 방지 : 트랜잭션이 커밋되어 확정된 데이터만 읽는 것을 허용
    - 대부분의 DBMS가 기본모드로 채택하고 있는 일관성 모드
    - Non-Repeatable Read, Phantom Read 현상은 여전히 발생
    - Oracle은 Lock을 사용하지 않고 쿼리시작 시점의 Undo 데이터를 제공하는 방식으로 구현
- 레벨 2 = `Repeatable Read`
    - 선생 트랜잭션이 읽은 데이터는 트랜잭션이 종료될 때까지 후행 트랜잭션이 갱신하거나 삭제하는 것을 불허함으로써 같은 데이터를 두 번 쿼리했을 때 일관성 있는 결과를 리턴
    - Phantom Read 현상은 여전히 발생
    - Oracle은 이 레벨을 명시적으로 지원하지 않지만 `for update절`을 이용해 구현 가능
- 레벨 3 = `Serializable Read`
    - 선행 트랜잭션이 읽은 데이터를 후행 트랜잭션이 갱신하거나 삭제하지 못할 뿐만 아니라 중간에 새로운 레코드를 삽입하는 것도 막아줌
    - 완벽한 읽기 일관성 모드를 제공
- 낮은 단계의 트랜잭션 고립화 수준을 사용하면 아래 3가지 현상이 발생한다.

### 2-3. Dirty Read(= Uncommitted Dependency)

- 아직 커밋되지 않은 수정 중인 데이터를 다른 트랜잭션에서 읽을 수 있도록 허용할 때 발생한다.
- 쓰기와 읽기가 충돌함으로서 일관성을 해치는 현상을 말한다.

### 2-4. Non-Repeatable Read(= Inconsistent Analysis)

- 한 트랜잭션 내에서 같은 쿼리를 두 번 수행할 때, 그 사이에 다른 트랜잭션 값을 수정 또는 삭제함으로써 두 쿼리의 결과가 상이하게 나타는 비일관성이다.

### 2-5. Phantom Read

- 한 트랜잭션 안에서 일정범위의 레코드들을 두 번 이상 읽을 때, 첫번째 쿼리에서 없던 유령 레코드가 두 번째 쿼리에서 나타나는 현상이다.

> 오라클은 트랜잭션 고립화 수준을 높이더라도 Lock을 사용하지 않으므로 동시성이 저하되지 않는다.
> 

→ Lock을 걸면 접근 제한으로 인해 성능이 저하된다.

→ Oracle에서는 이를 방지하기 위해 Lock을 쓰지 않는다. DeadLock에 걸릴 수 있기 때문이다.

→ 오라클에서는 Latch를 사용하여 다중 처리를 구현한다. Latch는 메모리에 읽기를 위한 대상을 올려두고 대상을 조작한다.
→ 이런 이유로 오라클은 완벽한 문장 수준의 읽기 일관성을 보장하지만, 트랜잭션에 대해서는 기본적으로 보장하지 않는다.

→ 관련 [링크](https://loosie.tistory.com/525)

---

## 3. 비관적 vs 낙관적 동시성 제어 ( Pessimistic VS Optimistic )

### 3-1. 비관적 동시성 제어

- 비관적 동시성 제어는 사용자들이 같은 데이터를 동시에 수정할 것이라고 가정한다.
- 한 사용자가 데이터를 읽는 시점에 Lock을 걸고 조회 또는 갱신처리가 완료될 때까지 이를 유지한다.
- Locking은 첫 번째 사용자가 트랜잭션을 완료하기 전까지 다른 사용자들이 그 데이터를 수정할 수 없다. → 이게 무한정으로 걸리면 문제가 되는 것
- 비관적 동시성 제어 방법
    - for update 사용 ( nowait, wait 옵션 )
        
        ```sql
        select 적립포인트, 방문횟수, 최근방문일시, 구매실적 from 고객
        where 고객번호 = :cust_num for update;
        ```
        
    - nowait와 wait 옵션은 Lock이 걸린 트랜잭션에 접근시 Lock을 얼마나 기다릴지다.
        
        ```sql
        for update nowait --> 대기 없이 Exception(ORA-00054)를 던짐
        for update wait 3 --> 3초 대기 후 Exception(ORA-30006)을 던짐
        ```
        
    - Lock이 걸린 `트랜잭션이 풀리지 않으면` 여전히 문제는 남아있다.

### 3-2. 낙관적 동시성 제어

- 사용자들이 같은 데이터를 동시에 수정하지 않을 것이라고 가정한다.
- 따라서 테이블을 읽을 때는 Lock을 설정하지 않는다.
- 데이터를 수정하고자 하는 시점에 앞서 읽은 데이터가 변경이 있었는지 확인한다.
- 충돌이 발생하면 rollback을 수행한다.
- 낙관적 동시성 제어 방법
    - 변경 시점을 확인하기 위해서 오라클 10g 이상 부터는 ORA_ROWSCN을 사용할 수 있다.
        
        ```sql
        select e.empno, e.ename, ORA_ROWSCN, SCN_TO_TIMESTAMP(ORA_ROWSCN)
        from emp e;
        ```
        
    - 이를 위해서는 테이블 생성 시 ROWDEPENDENCIES 옵션을 주어 생성해야 한다.
        
        ```sql
        create table t
        ROWDEPENDENCIES
        as
        select * from scott.emp;
        ```
        
    - 예시 → ORA_ROWSCN을 사용해서 데이터 변경을 확인한다.
        
        ```sql
        select 적리포인트, 방문횟수, 최근방문일시, 구매실적, ora_rowscn
        into :a, :b, :c, :d, :rowscn
        from 고객
        where 고객번호 = :cust_num;
        
        -- 새로운 적립포인트 계산
        update 고객 set 적립포인트 = :적립포인트
        where 고객번호 = :cust_num
        and ora_rowscn = :rowscn;
        ```
        

## 4. 동시성 구현 사례

## 5. 오라클 Lock
