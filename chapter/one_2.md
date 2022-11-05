### 1-1-3. 버퍼 Lock

![Untitled (8)](https://user-images.githubusercontent.com/91730236/193792737-44e2d628-664b-44d7-8842-a78d1a6d41e6.png)

1. 버퍼 Lock이란?
    1. 버퍼 캐시에 존재하는 오브젝트의 제어권한을 얻기 위한 잠금장치이다.
2. 버퍼 핸들
    1. 버퍼 헤더에 Pin을 설정하여고 사용하는 오브젝트를 ‘버퍼 핸들’이라고 부르며, 버퍼 핸들을 얻어 버퍼 헤더에 있는 소유자 목록에 연결시키는 방식으로 Pin을 설정함.
    2. 버퍼 헤더도 공유된 리소스이다.
    3. cache buffer handles 래치를 사용하여 버퍼 핸들을 얻음.
3. 버퍼 Lock의 필요성
    1. 버퍼 Pinning을 통한 블록 I/O 감소효과는 SQL 튜닝을 하는데 있어 중요함.
4. 버퍼 Pinning
    1. 버퍼를 읽고나서 버퍼 Pin을 즉각 해제하지 않고 데이터베이스 Call이 진행되는 동안 유지하는 기능을 말함.
    2. 버퍼 Pinning은 하나의 데이터베이스 Call(Parse Call, Execute, Call, Fetch Call) 내에서만 유효함 → Call이 끝나면 pin이 해제 되어야 함.


[버퍼 pin과 버퍼 lock 차이](https://support.oracle.com/knowledge/Oracle%20Cloud/444560_1.html#aref_section22)
- pin은 캐시(메모리 영역)에 동시성 관리
- lock은 process에 대한 동시성 관리
- [OREILLY 자료](https://www.oreilly.com/library/view/oracle-internals-an/156592598X/ch04s05.html)
- [buffer pin means](https://community.oracle.com/tech/developers/discussion/4326543/what-does-buffer-pin-mean)

### 1-1-4. Redo

1. Redo 로그는 데이터파일과 컨트롤 파일에 가해지는 모든 변경사항을 기록하기 위한 목적이다.
2. Redo 로그는 Online Redo와 Archived Redo 로그로 구성된다.
    1. Online Redp 로그는 Redo 로그 버퍼에 버퍼링된 로그 엔트리를 기록하는 파일로서 최소 2개 이상의 파일로 구성
    2. Archived Redo 로그는 Online Redo 로그가 재사용되기 전에 다른 위치로 백업해 둔 파일
3. 사용 목적
    1. Database Recovery
        1. 물리적으로 디스크가 깨지는 등의 Media Fail 발생 시 데이터베이스를 복구하기 위해 사용됨.
        2. Archived Redo 로그를 이용함
    2. Cache Recovery
        
        ![cacheRecovery drawio (1)](https://user-images.githubusercontent.com/91730236/193793082-95a38bc2-398d-4bda-9914-0e83bf72970a.png)
        
        1. Cache Recovery를 위해 사용됨.
        2. 트랜잭션 목적으로 캐시에 저장된 데이터가 물리적인 장애로 인해 휘발되는 경우 이를 복구하기 위해서 사용 됨.
    3. Fast Commit
        
        <img width="538" alt="Untitled (9)" src="https://user-images.githubusercontent.com/91730236/193793226-5156ae53-fe4e-4110-bd28-04cb0a647e89.png">
        
        1. 트랜잭션이 발생 시 건건이 데이터 파일에 기록하기보다 운선 변경사항을 Append 방식으로 빠르게 로그 파일에 기록하고 메모리 데이터 블록과 데이터 파일 간 동기화는 적절한 수단을 이용해 나중에 배치 방식으로 일괄 수행한다.
        2. 사용자의 갱신 내용이 메모리상의 버퍼 블록에만 기록된 채 아직 디스크에 기록되지 않았지만 Redo 로그를 믿고 빠르게 커밋을 완료한다는 의미에서 Fast Commit이라고 함.
        3. Crash가 발생하더라도 위 3-b의 도식처럼 복구가 되니까 괜츈하다는 로직이다.
        4. 이를 구현함에 있어 oracle에서는 ‘Delayed 블록 클린아웃’을 구현하여 사용한다.
4. Redo 레코드를 기록할 때도 곧바로 Redo 로그 버퍼에 기록 후 Redo log 파일에 저장한다. 일정 시점마다 LGWR가 Redo 로그 버퍼를 Redo 로그에 기록한다.
    1. LGWR가 Redo 로그 버퍼를 Redo 로그에 기록하는 시점
        1. 3초마다 DBWR 프로세스로부터 신호를 받을 때
        2. 로그 버퍼의 1/3이 차거나 기록된 Redo 레코드량이 1MB를 넘을 때
        3. 사용자가 커밋 또는 롤백 명령을 날릴 때



`[UNDO와 REDO]`
- 커밋을 한 상황에서 디스크에 반영이 안될 수 있다. 커밋 정보가 캐시에만 있으면(Fast Commit)
- 디스크도 손상이 되었다면 archive redo log file로 복구
- 비정상 조료로 캐시만 날라갔을 때 redo log로 fast commit까지도 복구 가능
- 미처 commit 되지 않은 transaction은 `undo`로 롤백

Q&A
- Redo 전체 복구하고 싶지 않을 때? 특정 check point까지만 복구하고 싶을때? -> 엔지니어가 수동으로 제어를 하던지 AUM에서 관리를 한다고 되어 있음.
- Redo까지 망가져 있으면 -> 노답 

![Untitled (12)](https://user-images.githubusercontent.com/91730236/193847599-633f4324-4ca3-4f85-873d-94aad49bfd7f.png)




### 1-1-5. Undo(Rollback)

<aside>
🔥 undo = Rollback 이다.

</aside>

1. Undo 세그먼트 트랜잭션 테이블 슬롯
    1. Undo 세그먼트 : 롤백하거나 실행 취소하는데 사용하는 정보를 생성하고 관리하기 위한 세그먼트
    2. Undo Segment 사용 목적 
        1. Transaction Rallback
        2. Transaction Recovery
        3. Read Consistency
    3. segment 구조
        
        <img width="525" alt="Untitled (10)" src="https://user-images.githubusercontent.com/91730236/193793736-6f7eb46f-46a6-41fe-8c38-f36284c4f2b4.png">

        
    4. 각 데이터 구성
        1. Transaction Table Slot in Undo Segment Header
            1. 트랜잭션 ID
            2. 트랜잭션 상태정보
            3. 커밋 SCN
            4. Last UBA
            5. 기타
2. 블록 헤더 ITL 슬롯
    1. 구조
        
        <aside>
        🔥 ITL(Interested Transaction List) - Block Header 내부 Transaction 정보를 관리하는 List
        
        </aside>
        
        <img width="519" alt="Untitled (11)" src="https://user-images.githubusercontent.com/91730236/193793989-e9ce7644-4035-4fe6-b33e-d8d3d7b922e5.png">

        
    2. ITL 슬롯에 저장되는 데이터
        1. ITL 슬롯 번호
        2. 트랜잭션 ID
        3. UBA
        4. 커밋 Flag
        5. Locking 정보
        6. `커밋 SCN`
3. Lock Byte
    1. 레코드가 저장되는 로우마다 그 헤더에 Lock Byte를 할당해 해당 로우를 갱신 중인 트랜잭션 ITL 슬롯 번호를 기록해 둔다 → Row-Level Lock
    2. 오라클은 로우 단위 Lock과 트랜잭션 Lock을 조합해서 로우 Lock을 구현함.

Q&A.
```
ITL에 있는 커밋 SCN과 Undo에 있는 커밋 SCN이 다를 수 있다. -> 언제 다를 수 있나?
Undo 세그먼트 정보와 개별 블록정보와는 불일치가 있을 수 있다.
```

### 1-1-6. 문장수준 읽기 일관성

<aside>
🔥 읽기 일관성 : 트랜잭션 내에서 SQL문이 차례로 수행되는 도중에 다른 트랜잭션에 의해 데이터가 변경, 추가, 삭제되는 현상을 방지하기 위한 장치

</aside>

1. 문장수준 읽기 일관성이란?
    1. 단일 SQL문이 수행되는 도중에 다른 트랜잭션에 의해 데이터의 추가, 변경, 삭제가 발생하더라도 일관성 있는 결과집합을 리턴하는 것
    2. 읽기 작업에 대해 Shared Lock을 사용함으로써 Exclusive Lock이 걸린 로우를 읽지 못하도록 한다.
    3. 즉 Dirty Read를 허용하지 않겠다.
2. Consistent 모드 블록 읽기
3. Consistent 모드 블록 읽기의 세부원리


`오라클은 왜 Undo Segment를 이용하여 문장수준 읽기 일관성을 보장할까?`
- lock 매커니즘 : `shared` VS `exclusive`
    - shared lock : read lock
    - exclusive lock : write lock
- DBMS간 비교
    - 오라클, mysql 진영 : undo 세그먼트 이용 방식 -> 동시성 확보가 중요 -> OLAP에 용이함
    - PostgreSQL, Firebase : record의 멀티 버져닝 기록 & 관리 -> 동시성 확보가 중요 -> OLTP에 용이함
    - 근데 왜 이렇게 하는것인가?


### 1-1-7. Consistent vs Current 모드 읽기

1. Consistent, Current 모드
    1. Consistent : SCN 확인 과정을 거치며 쿼리가 시작된 시점을 기준으로 일관성 있는 상태로 블록을 액세스 하는 것
    2. Current : SQL문이 시작된 시점이 아니라 데이터를 찾아간 바로 그 시점의 최종 값을 읽으려고 블록을 액세스 하는 것
2. 책에서 명시하는 문제점들은 결국 두 읽기 모드의 차이가 데이터에 접근하는 차이를 만들어 내기때문에 문제가 발생 할 수 있음

### 1-1-8. 블록 클린 아웃

<aside>
🔥 블록 클린아웃 : 트랜잭션에 의해 설정된 로우 Lock을 해제하고 블록 헤더에 커밋 정보를 기록하는 오퍼레이션

</aside>

1. 블록 클린아웃 동작 시점 : 해당 블록이 처음 액세스 되는 시점.
2. 동작 매커니즘
    1. Delayed 블록 클린아웃 → **전체 캐시 블록 중 일정이상 차면 비워내겠다.**
        1. 트랜잭션이 갱신한 블록 개수가 총 버퍼 캐시 블록 개수의 1/10을 초과할 때 사용하는 방식
        2. 커밋 이후 해당 블록을 액세스하는 첫 번째 쿼리에 의해 클린아웃이 이루어지며, 다음 작업 수행
            1. ITL 슬롯에 커밋 정보 저장
            2. 레코드에 기록된 Lock Byte 해제
            3. Online Redo에 Logging
    2. 커밋 클린아웃(= fast 블록 클린아웃)
    3. ITL과 블록 클린아웃

### 1-1-9. Snapshot too old

<aside>
🔥 Snapshot too old는 오라클에서 발생하는 에러다.
쿼리 수행시간이 오래 걸려서 발생하는 undo 관련 에러

</aside>

1. 발생원인
    1. 데이터를 읽어 내려가다가 쿼리 SCN 이후에 변경된 블록을 만나 과거 시점으로 롤백한 ‘Read Consistent’ 이미지를 얻으려고 하는데, Undo Block이 다른 트랜잭션에 의해 이미 재사용되어 필요한 Undo 정보를 얻을 수 없는 경우.
    2. 커밋된 트랜잭션 테이블 슬롯이 다른 트랜잭션에 의해 재사용되어 커밋 정보를 확인할 수 없는 경우.
2. 회피 방법
    1. 불필요한 커밋을 자주 수행하지 않는다.
    2. fetch cross commit 형태의 프로그램 작성을 피해 다른 방식으로 구현한다.
    3. 트랜잭션이 몰리는 시간대에 오래 걸리는 쿼리가 같이 수행되지 않도록 시간을 조정한다.
    4. 큰 테이블을 일정 범위로 나누어 읽고 단계적으로 실행할 수 있도록 코딩한다.
    5. 오랜 시간에 걸쳐 같은 블록을 여러 번 방문하는 Nested Loop형태의 조인문 또는 인덱스를 경유한 테이블 액세스를 수반하는 프로그램이 있는지 체크하고, 이를 회피 할수 있는 방법을 찾는다.
    6. 소트 부하를 감수하더라도 order by 등을 강제로 삽입해 소트연산이 발생하도록 한다.
    7. 만약 delayed 블록 클린아웃에 의해 Snapshot too old가 발생하는 것으로 의심되면 대량 업데이트 후에 곧바로 해당 테이블에 대해 Full Scan하도록 쿼리를 날리는 것도 하나의 해결 방법이 될 수 있다.

### 1-1-10. 대기 이벤트

<aside>
🔥 대기 이벤트 : 오라클 프로세스가 작업을 수행 할 조건이 충족되기까지 sleep 상태가 되는 것

</aside>

1. 대기 이벤트가 발생하는 경우
    1. 자신이 필요로 하는 특정 리소스가 다른 프로세스에 의해 사용 중일 때
    2. 다른 프로세스에 의해 선행작업이 완료되기를 기다릴 때
    3. 할 일이 없을 때
2. 대기 이벤트는 언제 사라질까?
    1. 대기 상태에 빠진 프로세스가 기다리던 리소스가 사용 가능해 지거나
    2. 작업을 계속 진행하기 위한 선행작업이 완료되거나
    3. 해야 할 작업이 있을 경우

### 1-1-11. Shared Pool

<aside>
🔥 Data Dictionary : 대부분 읽기 전용으로 제공되는 테이블 및 뷰들의 집합

</aside>

1. 딕셔너리 캐시
    1. 오라클 딕셔너리 정보를 저장해두는 캐시영역
    2. 테이블, 인덱스와 같은 오브젝트뿐만 아니라 테이블스페이스, 데이터파일, 세그먼트, 익스텐트, 사용자, 제약, 시퀀스, DB Link에 관한 정보들을 캐싱한다.
2. 라이브러리 캐시
    1. 사용자가 던진 SQL과 그 실행계획을 저장해두는 캐시영역
    2. SQL 최적화 과정에서 하드파싱을 최소화하기 위한 캐시 → 최적화 된 SQL을 저장하는 캐시

---
