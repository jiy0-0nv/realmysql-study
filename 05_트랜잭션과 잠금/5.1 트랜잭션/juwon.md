# 3회차 : 5.1 트랜잭션 ~ 5.3 InnoDB 스토리지 엔진 잠금

이번 5장에서는 MySQL의 동시성에 영향을 미치는 트랜잭션, 잠금, 그리고 트랜잭션 격리 수준을 다룬다.

## 5.1. 트랜잭션

### 5.1.1. MySQL에서의 트랜잭션

#### 개념
- 작업의 완전성 보장: 논리적 작업 셋을 모두 수행하거나, 전혀 수행하지 않도록 보장
- 목적: 처리하지 못할 경우에는 원 상태로 복구해서 작업의 일부만 적용되는 Partial Update 현상 방지
- '잠금' vs. '트랜잭션'
  - 잠금: 여러 커넥션이 동시에 동일 자원 변경 시 순서 제어 (동시성 제어)
  - 트랜잭션: 일부만 적용되는 것을 방지해 데이터 정합성 유지 (데이터 정합성 보장)

---

#### MyISAM vs InnoDB 예제
```sql
-- MyISAM
CREATE TABLE tab_myisam (fdpk INT PRIMARY KEY) ENGINE=MyISAM;
INSERT INTO tab_myisam (fdpk) VALUES (3);

-- InnoDB
CREATE TABLE tab_innodb (fdpk INT PRIMARY KEY) ENGINE=InnoDB;
INSERT INTO tab_innodb (fdpk) VALUES (3);

SET autocommit=ON;

INSERT INTO tab_myisam (fdpk) VALUES (1),(2),(3); -- 부분 적용
INSERT INTO tab_innodb (fdpk) VALUES (1),(2),(3); -- 전체 롤백
```
- 결과
  - MyISAM: 오류 발생해도 `1, 2` 저장됨 -> Partial Update
  - InnoDB: 트랜잭션 원칙에 따라 전체 롤백

---

### 5.1.2 주의사항

#### 트랜잭션 범위 최소화
- 커넥션 점유 시간 최소화
- 네트워크/외부 작업 배제
- DB 작업끼리만 트랜잭션 묶기

#### 개선 전
```plaintext
1) 처리 시작
    => 데이터베이스 커넥션 생성
    => 트랜잭션 시작
2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장
5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
7) 저장된 내용 또는 기타 정보를 DBMS에 조회
8) 게시물 등록에 대한 알림 메일 발송
9) 알림 메일 발송 이력을 DBMS에 저장
    <= 트랜잭션 종료(COMMIT)
    <= 데이터베이스 커넥션 반납
10) 처리 완료
```

#### 문제점 파악
- 트랜잭션 범위 최소화
   - 실제 DB 저장 작업(트랜잭션)은 5번부터 시작
   - 로그인 확인(2번), 입력 검증(3번), 파일 저장(4번) 등은 트랜잭션에 포함되면 안됨
   - 커넥션을 오래 점유하면 여유 커넥션이 줄어들고, 대기 상황이 발생할 수 있음
 - 네트워크/외부 작업 배제
   - 메일 전송, FTP 전송 등 네트워크 작업은 트랜잭션 내부에서 제외해야 함
   - 네트워크 장애 시 웹 서버뿐 아니라 DBMS까지 위험해질 수 있음
 - DB 작업 분리
   - 사용자 입력 저장(5번)과 첨부파일 메타 저장(6번)은 반드시 하나의 트랜잭션으로 묶음
   - 저장 데이터 조회(7번)은 트랜잭션 불필요
   - 메일 발송 이력 저장(9번)은 별도 트랜잭션으로 분리 가능

#### 개선 후
```plaintext
1) 처리 시작
2) 사용자의 로그인 여부 확인
3) 사용자의 글쓰기 내용의 오류 발생 여부 확인
4) 첨부로 업로드된 파일 확인 및 저장
    => 데이터베이스 커넥션 생성(또는 커넥션 풀에서 가져오기)
    => 트랜잭션 시작
5) 사용자의 입력 내용을 DBMS에 저장
6) 첨부 파일 정보를 DBMS에 저장
    <= 트랜잭션 종료(COMMIT)
7) 저장된 내용 또는 기타 정보를 DBMS에서 조회
8) 게시물 등록에 대한 알림 메일 발송
    => 트랜잭션 시작
9) 알림 메일 발송 이력을 DBMS에 저장
    <= 트랜잭션 종료(COMMIT)
    <= 데이터베이스 커넥션 종료(또는 커넥션 풀에 반납)
10) 처리 완료
```

---

### 5.1절 주요 포인트
* 트랜잭션은 작업의 완전성을 보장하며 Partial Update를 방지함
* MyISAM은 트랜잭션을 지원하지 않아 부분 적용 가능성이 있고, InnoDB는 오류 발생 시 전체 롤백함
* 트랜잭션 범위는 최소화하고, 네트워크·외부 작업은 트랜잭션 밖에서 수행해야 함

## 5.2. MySQL 엔진의 잠금

MySQL에서 사용되는 잠금 -> 스토리지 엔진 레벨 / MySQL 엔진 레벨
- MySQL 엔진 레벨 잠금
  - 모든 스토리지 엔진에 영향을 미침
  - ex. 글로벌 락, 테이블 락, 메타데이터 락, 네임드 락
- 스토리지 엔진 레벨 잠금
  - 해당 스토리지 엔진 내부에서만 작동
  - 엔진 간 상호 영향 없음 (ex. InnoDB 레코드 락)

### 5.2.1 글로벌 락
```sql
-- 글로벌 락
FLUSH TABLES WITH READ LOCK;

-- MySQL 8.0+ 백업 락(InnoDB의 일반화로 인한 경량화)
LOCK INSTANCE FOR BACKUP;
UNLOCK INSTANCE;
```
#### 개념
- MySQL에서 제공하는 가장 범위가 넓은 잠금
- 획득 시, SELECT를 제외한 모든 DML/DDL이 대기 상태로 전환
- 서버 전체에 적용되며, DB/테이블이 달라도 동일하게 영향

#### 사용 예시
- 여러 DB/테이블의 일관된 스냅샷 백업이 필요할 때  
  (ex. MyISAM, MEMORY 테이블을 mysqldump로 백업)

#### 주의점
- 장시간 SELECT가 실행 중이면, 글로벌 락이 그 쿼리 종료까지 대기
- 웹 서비스용 서버에서는 가급적 사용 지양
- MySQL 8.0에서는 백업 락(Backup Lock) 도입 → 글로벌 락보다 영향이 적음(스키마 변경, 계정 변경만 차단, DML 허용)

---

### 5.2.2. 테이블 락

#### 개념
- 특정 테이블 단위로 설정되는 잠금
- 명시적(LOCK TABLES) 또는 묵시적(엔진에 따라 자동)으로 발생

#### 명시적 잠금
```sql
-- 명시적 테이블 락
LOCK TABLES table_name READ;   -- 읽기 잠금
LOCK TABLES table_name WRITE;  -- 쓰기 잠금
UNLOCK TABLES;                 -- 해제
```
- MyISAM뿐 아니라 InnoDB도 잠금 가능
- DDL 변경 작업 시 InnoDB에서도 테이블 락이 적용됨

#### 묵시적 잠금
- MyISAM/MEMORY: DML 실행 시 자동 획득 -> 실행 후 즉시 해제
- Inno DB: 대부분 DML은 레코드 락 기반이므로 묵시적 테이블 락 없음 (단, 스키마 변경 시는 테이블 락 사용)

---

### 5.2.3 네임드 락
#### 개념
- 테이블, 레코드, 스키마가 아닌 임의의 문자열을 잠금 대상으로 사용
- 여러 클라이언트나 서버 간 동기화 제어에 사용

#### 사용 예시
```sql
-- 잠금 획득 (최대 2초 대기)
SELECT GET_LOCK('mylock', 2);

-- 잠금 상태 확인
SELECT IS_FREE_LOCK('mylock');

-- 잠금 해제
SELECT RELEASE_LOCK('mylock');
```

#### 특징
- 동기화가 필요한 프로그램/서버 간에 충돌 방지
- 대규모 배치 작업 시 데드락 회피용으로도 사용
- MySQL 8.0: 네임드락 중첩 획득 및 `RELEASE_ALL_LOCKS()` 지원

---

### 5.2.4. 메타데이터 락

#### 개념
- 테이블, 뷰, 스키마 등 객체의 이름이나 구조 변경 시 자동 획득
- ex. RENAME TABLE, ALTER TABLE

#### 사용 예시
```sql
-- 두 개의 RENAME 작업을 한 번에 -> 순간적인 테이블 부재 방지
RENAME TABLE rank TO rank_backup, rank_new TO rank;
```

#### 특징
- 하나의 명령문에서 여러 객체 변경 가능
- DDL 작업이 동시에 실행되면 충돌 가능
- 테이블 변경 시 해당 객체를 참조하는 모든 세션이 영향을 받을 수 있음

---

### 5.2절 주요 포인트
* MySQL 엔진 잠금은 글로벌 락, 테이블 락, 네임드 락, 메타데이터 락 등 네 가지 주요 형태가 있음
* 글로벌 락은 서버 전체에 영향을 미치므로 주의가 필요하며, MySQL 8.0부터는 백업 락으로 대체 가능
* 테이블 락과 네임드 락은 특정 테이블 또는 문자열 단위로 동기화를 제어하며, 메타데이터 락은 객체 구조 변경 시 자동 획득됨


## 5.3. InnoDB 스토리지 엔진 잠금

- InnoDB는 MySQL 엔진 레벨 잠금과는 별도로 스토리지 엔진 내부에서 레코드 기반 잠금(record-level locking)을 제공  
- 이 덕분에 MyISAM보다 동시성 처리가 훨씬 뛰어나지만, 내부 잠금 방식의 특성과 제약을 이해해야 성능 저하나 데드락을 방지할 수 있음

### 5.3.1. InnoDB 스토리지 엔진의 잠금

#### 1) 레코드 락 (Record Lock)
- 인덱스의 특정 레코드에만 잠금 설정
- 인덱스가 없는 경우에도 InnoDB는 내부적으로 생성된 클러스터 인덱스를 사용해 잠금
- 프라이머리 키 또는 유니크 인덱스를 통한 변경 시에는 해당 레코드만 잠금, 갭 잠금은 발생하지 않음

#### 2) 갭 락 (Gap Lock)
- 레코드와 레코드 사이의 간격에 잠금을 설정
- 해당 구간에 새로운 레코드가 INSERT 되는 것을 차단
- 단독으로 쓰이는 경우보다는 넥스트 키 락의 일부로 사용

#### 3) 넥스트 키 락 (Next Key Lock)
- 레코드 락 + 갭 락 결합 형태
- `REPEATABLE READ` 격리 수준에서 주로 사용
- STATEMENT 포맷의 바이너리 로그 사용 시, 레플리카 서버와 결과 일관성을 보장하기 위해 필요
- 그러나 불필요한 대기나 데드락 원인이 되기도 함 -> 가능하다면 ROW 포맷 바이너리 로그 사용 권장

#### 4) 자동 증가 락 (AUTO_INCREMENT Lock)
- `AUTO_INCREMENT` 컬럼 값 생성 시 테이블 단위로 짧게 걸리는 락
- INSERT/REPLACE 문장에서만 사용, UPDATE/DELETE에는 적용되지 않음
- MySQL 5.1 이상에서는 `innodb_autoinc_lock_mode`로 동작 방식 변경 가능
  - 0: 모든 INSERT에 자동 증가 락 사용
  - 1: 건수를 예측할 수 있는 경우 가벼운 래치(mutex) 사용 (연속 모드)
  - 2: 항상 래치 사용, 연속 값 보장 안 함 (인터리빙 모드, MySQL 8.0 기본값)

---

### 5.3.2. 인덱스와 잠금

InnoDB는 레코드 자체가 아니라 인덱스 레코드를 잠근다.  
변경 대상 레코드를 찾기 위해 사용한 인덱스의 모든 레코드가 잠금 대상이 된다.

#### 예시
```sql
-- first_name 칼럼만 인덱스가 있는 경우
UPDATE employees
SET hire_date = NOW()
WHERE first_name = 'Georgi' AND last_name = 'Klassen';
```
- first_name만 인덱스 적용 → first_name='Georgi'인 모든 레코드(예: 253건) 잠금
- 인덱스가 없으면 테이블 풀스캔 시 모든 레코드 잠금 → 동시성 크게 저하
- WHERE 절의 조건과 인덱스 설계가 잠금 범위를 결정

---

### 5.3.3. 레코드 수준의 잠금 확인 및 해제
MySQL 8.0에서는 `performance_schema`를 통해 현재 잠금 상태와 대기 관계를 확인할 수 있다.
```sql
SELECT r.trx_id waiting_trx_id, r.trx_mysql_thread_id waiting_thread,
       r.trx_query waiting_query, b.trx_id blocking_trx_id,
       b.trx_mysql_thread_id blocking_thread, b.trx_query blocking_query
FROM performance_schema.data_lock_waits w
JOIN information_schema.innodb_trx b
  ON b.trx_id = w.blocking_engine_transaction_id
JOIN information_schema.innodb_trx r
  ON r.trx_id = w.requesting_engine_transaction_id;
```
- `waiting_*`: 잠금을 기다리는 트랜잭션
- `blocking_*`: 잠금을 보유하고 있는 트랜잭션
- 필요 시 `KILL <thread_id>`로 강제 해제 -> 특정 스레드를 종료하면 해당 스레드가 보유한 잠금도 해제됨

---

### 5.3절 주요 포인트
* InnoDB 잠금은 인덱스 설계, 격리 수준, 바이너리 로그 포맷 등에 따라 동작 방식이 달라짐
* 인덱스를 효율적으로 설계하지 않으면 불필요하게 많은 레코드에 잠금이 걸려 동시성이 떨어질 수 있음
* 잠금 모니터링 및 대기 관계 파악은 performance_schema로 가능

---

## 3회차 주요 내용
1. MyISAM vs InnoDB 트랜잭션 처리 차이
2. 잠금 범위와 영향 최소화
3. InnoDB 인덱스 설계 중요성
4. 서비스 성격에 맞는 격리 수준 선택