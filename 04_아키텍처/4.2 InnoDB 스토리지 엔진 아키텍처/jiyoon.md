# 4.2 InnoDB 스토리지 엔진 아키텍처

## 4.2.1 프라이머리 키에 의한 클러스터링  
모든 테이블을 프라이머리 키 순서대로 디스크에 클러스터링해 저장한다.  
- 클러스터링 인덱스  
  - 레코드의 물리적 저장 순서를 결정
  - 세컨더리 인덱스는 레코드 주소 대신 프라이머리 키 값을 논리 주소로 사용
- 범위 스캔 성능 향상  
  - 프라이머리 키 기반 레인지 스캔이 매우 빠름  
  - 옵티마이저가 보조 인덱스보다 프라이머리 키를 우선 선택하도록 유도함  
- MyISAM과의 차이  
  - MyISAM은 클러스터링 키를 지원하지 않아 프라이머리·세컨더리 인덱스가 동일한 구조를 가짐  

## 4.2.2 외래 키 지원  
외래 키 제약을 엔진 레벨에서 관리해 부모·자식 테이블 간 무결성을 보장한다.
- 인덱스 요구 조건  
  - 외래 키 칼럼에 반드시 인덱스가 있어야 함  
- 잠금 전파  
  - 부모·자식 테이블 변경 시 자동으로 잠금이 전파되어 데드락 위험이 높아짐  
- 검사 일시 중단  
    | 설정 방법                                | 효과                                   |
    | ---------------------------------------- | -------------------------------------- |
    | `SET SESSION foreign_key_checks=OFF;`    | 외래 키 검사 일시 중단 (세션 단위)     |
    | `SET GLOBAL foreign_key_checks=OFF;`     | 외래 키 검사 전역 중단 (서버 단위)     |

## 4.2.3 MVCC (Multi-Version Concurrency Control)  
언두 로그를 통해 다중 버전 일관 읽기를 제공한다.
- 버전 관리  
  - 각 레코드에 대해 변경 전·후 버전을 모두 유지해 잠금 없이 읽기 가능  
- 격리 수준별 동작  
  - READ COMMITTED  
    - 커밋된 최신 값을 읽음  
  - REPEATABLE READ 이상  
    - 언두 로그의 이전 버전을 반환해 동일 트랜잭션 내 일관성 유지  
- 언두 로그 관리  
  - 오래 열린 트랜잭션이 언두 로그 공간을 차지하므로 가능한 한 빨리 종료해야 함  

## 4.2.4 잠금 없는 일관된 읽기 (Non-Locking Consistent Read)  
MVCC를 활용해 SELECT 시 레코드 잠금을 기다리지 않고 즉시 실행한다. 
- 격리 수준 조건  
  - SERIALIZABLE이 아니면 변경 중인 레코드도 언두 로그나 버퍼 풀에서 일관된 데이터를 읽음  
- 언두 로그 영향  
  - 장시간 열려 있는 읽기 트랜잭션이 언두 로그를 유지해 시스템 성능 저하 유발  
- 권장 사항  
  - 읽기 전용 작업도 빠르게 완료하고 커밋·롤백 처리해야 함  

## 4.2.5 자동 데드락 감지  
잠금 대기 그래프를 주기적으로 검사해 데드락을 탐지·해제한다.
- 감지 메커니즘  
  - 데드락 감지 스레드가 대기 그래프를 순회해 교착 상태 여부 확인  
  - 교착 상태인 트랜잭션 중 언두 로그 양이 적은 쪽을 롤백
- 설정 옵션  
  - `innodb_deadlock_detect=OFF`  
    - 감지 스레드를 비활성화하고 `innodb_lock_wait_timeout`으로 대기 시간을 제한함  
- 부하 고려  
  - 동시 처리 스레드가 많을 때 감지 스레드가 병목이 될 수 있어 설정 조정 필요  

## 4.2.6 자동화된 장애 복구  
서버 시작 시 손상된 트랜잭션과 부분 기록된 페이지를 자동 복구한다.
- 강제 복구 모드  
  - `innodb_force_recovery` 값을 1~6 단계별로 높여가며 복구 시나리오 적용  
- 복구 단계별 동작  
  - 낮은 단계  
    - 언두 로그와 리두 로그 검사 및 적용  
  - 높은 단계  
    - 백그라운드 스레드 비활성화, 로그 무시 등 더욱 강력한 복구 시도  
- 후속 조치  
  - 복구 완료 후 `mysqldump`로 데이터 백업하고 서버 재구축 권장  

## 4.2.7 InnoDB 버퍼 풀  
버퍼 풀은 디스크 I/O를 줄이고 랜덤 쓰기 성능을 높이는 핵심 캐시 메커니즘이다.
- 크기 및 인스턴스  
  - `innodb_buffer_pool_size`: 전체 크기 설정
    - 메모리 상황에 따라 50~80% 범위로 조정
  - `innodb_buffer_pool_instances`: pool 분할 -> 내부 잠금 경합 완화
- 캐시 구조  
  - 페이지 단위로 LRU 리스트·플러시 리스트·프리 리스트를 운영
  - 자주 사용하는 페이지는 LRU 리스트 상단에 유지되고, 덜 사용하는 페이지는 제거되거나 디스크로 플러시
- 덤프/로드 자동화  
  - `innodb_buffer_pool_dump_at_shutdown`  
    - 셧다운 시 버퍼 풀 상태 덤프
  - `innodb_buffer_pool_load_at_startup`  
    - 시작 시 덤프된 상태를 로드

## 4.2.8 Double-Write 버퍼  
파셜 페이지(partial-page) 또는 톤 페이지(torn-page) 발생을 방지하기 위해 Double-Write 기법을 사용한다.
- 동작 원리  
  - ‘A’~‘E’ 더티 페이지를 하나로 묶어 시스템 테이블스페이스의 Double-Write 버퍼에 순차 쓰기 수행 :contentReference[oaicite:6]{index=6}  
  - 이후 각 페이지를 실제 데이터 파일의 랜덤 위치에 개별 쓰기 실행  
  - 재시작 시 데이터 파일과 버퍼를 비교해 불일치 페이지는 버퍼 내용으로 복구  
- 설정  
  - `innodb_doublewrite=ON` (기본 활성화)  
  - SSD 환경에서 랜덤 I/O 비용이 중요하면 OFF 고려하나, 리두 로그 동기화(`innodb_flush_log_at_trx_commit≠1`) 설정 시 함께 비활성화 권장 :contentReference[oaicite:7]{index=7}  
- 운영 팁  
  - HDD에서는 부담 작음  
  - 데이터 무결성이 중요한 서비스는 항상 활성화  

## 4.2.9 언두 로그(Undo Log)  
 트랜잭션 롤백과 격리 수준 보장을 위해 변경 전 데이터를 언두 로그로 백업한다. 
- 용도  
  - 롤백 시 변경 전 데이터 복구  
  - MVCC를 통한 일관된 읽기 제공  
- 관리 이슈  
  - 장시간 활성 트랜잭션이 언두 로그를 보존해 디스크 사용량 증가 및 조회 성능 저하 유발 :contentReference[oaicite:8]{index=8}  

### 언두 로그 모니터링  
  - `SHOW ENGINE INNODB STATUS\G` 의 TRANSACTIONS → History list length 확인  
  - MySQL 8.0+:  
    ```sql
    SELECT COUNT(*) 
      FROM information_schema.innodb_metrics 
     WHERE subsystem='transaction' 
       AND name='trx_rseg_history_len';
    ```  

### 언두 테이블스페이스 관리  
  - MySQL 5.6 이전: 언두 로그는 시스템 테이블스페이스(ibdata)에만 저장  
  - MySQL 5.6+:  
    - `innodb_undo_tablespaces>0` 설정 시 별도 언두 테이블스페이스 사용  
    - MySQL 8.0 이후 항상 외부 로그 파일 사용 (Deprecated: 해당 변수) :contentReference[oaicite:9]{index=9}  
  - 구조  
    - 언두 테이블스페이스는 1~128 롤백 세그먼트 보유  
    - 각 세그먼트는 페이지 크기/16 개수만큼 언두 슬롯 보유  
  - 동적 관리  
    ```sql
    -- 테이블스페이스 추가
    CREATE UNDO TABLESPACE extra_undo_003 
       ADD DATAFILE '/data/undo_003.ibu';
    -- 비활성화 및 삭제
    ALTER UNDO TABLESPACE extra_undo_003 SET INACTIVE;
    DROP UNDO TABLESPACE extra_undo_003;
    ```  
  - 언두 로그 자동 축소  
    - 자동 모드 (`innodb_undo_log_truncate=ON`): 퍼지 스레드가 주기적 언두 로그 잘라내기 수행  
    - 수동 모드: 테이블스페이스 비활성화 후 퍼지 스레드가 비활성 공간 반납  

## 4.2.10 체인지 버퍼(Change Buffer)  
 디스크에서 인덱스 페이지를 읽어오기 전 인덱스 변경 작업을 임시 저장해 성능을 향상시키는 버퍼링 기능을 제공한다.

- 특징  
  - 버퍼 풀에 없는 인덱스 변경(`INSERT`, `DELETE`, `UPDATE`) 시 즉시 디스크 I/O 없이 Change Buffer에 저장 :contentReference[oaicite:10]{index=10}  
  - 유니크 인덱스는 중복 체크 필요해 미지원  
  
- 백그라운드 병합  
  - Merge thread가 Change Buffer → 실제 인덱스 페이지 병합 수행 
   
- 설정 (`innodb_change_buffering`)  
  | 값      | 버퍼링 대상                         |
  |---------|-------------------------------------|
  | all     | inserts + deletes + purges          |
  | none    | 버퍼링 안함                         |
  | inserts | 신규 아이템 추가만                  |
  | deletes | 삭제 표시만                         |
  | changes | inserts + deletes                   |
  | purges  | 백그라운드 purges만                 |

- 모니터링  
  ```sql
  SELECT EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED
    FROM performance_schema.memory_summary_global_by_event_name
   WHERE EVENT_NAME='memory/innodb/ibuf0ibuf';
   ```

## 4.2.11 리두 로그(Redo Log) 및 로그 버퍼  
Redo Log는 트랜잭션의 내구성을 보장하기 위해 변경 내용을 먼저 기록한다.
- 역할  
  - 비정상 종료 시 커밋된 변경 복구  
  - 언두 로그와 함께 롤백 여부 확인용으로 사용  
- 주요 설정  
  | 변수                              | 동작 방식                                               |
  |-----------------------------------|---------------------------------------------------------|
  | `innodb_flush_log_at_trx_commit=0` | 1초마다 write+sync 수행 → 최대 1초 변경 손실 가능       |
  | `=1` (권장)                       | 커밋 시마다 write+sync 수행 → 변경 손실 없음            |
  | `=2`                              | 커밋 시 write 만 수행, 1초마다 sync 수행 → OS 버퍼 활용 |
- 로그 버퍼  
  - `innodb_log_buffer_size` (기본 16MB)  
  - 대용량 BLOB/TEXT 변경 시 버퍼 크기 증가 권장  

### 리두 로그 아카이빙  
MySQL 8.0부터 제공되는 아카이빙 기능을 통해 Redo Log를 외부 저장소에 보관

- 초기 설정  
  ```bash
  mkdir -p /var/log/mysql_redo_archive/$(date +%Y%m%d)
  chmod 700 /var/log/mysql_redo_archive/$(date +%Y%m%d)
  ```

- 아카이브 시작
    ```
    SET GLOBAL innodb_redo_log_archive_dirs='backup:/var/log/mysql_redo_archive'
    DO innodb_redo_log_archive_start('backup',''"$(date +%Y%m%d)"'')
    ```

- 아카이브 종료
    ```
    DO innodb_redo_log_archive_stop()
    ```

## 4.2.11 리두 로그(Redo Log) 및 로그 버퍼  
Redo Log는 트랜잭션 내구성을 보장하기 위해 변경 내용을 먼저 기록함  
- 역할  
  - 비정상 종료 시 커밋된 변경 복구  
  - 언두 로그와 함께 롤백 여부 확인용으로 사용  
- 주요 설정  
  | 변수                                    | 동작 방식                                               |
  |---------------------------------------|---------------------------------------------------------|
  | `innodb_flush_log_at_trx_commit=0`    | 1초마다 write+sync 수행 → 최대 1초 변경 손실 가능      |
  | `innodb_flush_log_at_trx_commit=1` (권장) | 커밋 시마다 write+sync 수행 → 변경 손실 없음           |
  | `innodb_flush_log_at_trx_commit=2`    | 커밋 시 write만 수행, 1초마다 sync 수행 → OS 버퍼 활용 |
- 로그 버퍼  
  - `innodb_log_buffer_size` (기본 16MB)  
  - 대용량 BLOB/TEXT 변경 시 버퍼 크기 증가 권장  

### 리두 로그 아카이빙  
- MySQL 8.0부터 제공되는 기능
- Redo Log를 외부 저장소에 보관  

- 초기 설정  
  - `mkdir -p /var/log/mysql_redo_archive/$(date +%Y%m%d)`  
  - `chmod 700 /var/log/mysql_redo_archive/$(date +%Y%m%d)`
    
- 아카이브 시작  
  - `SET GLOBAL innodb_redo_log_archive_dirs='backup:/var/log/mysql_redo_archive'`  
  - `DO innodb_redo_log_archive_start('backup',''"$(date +%Y%m%d)"'')`  

- 아카이브 종료  
  - `DO innodb_redo_log_archive_stop()`  

### 리두 로그 활성화/비활성화  
- MySQL 8.0부터 Redo Log를 수동으로 제어 가능

- 비활성화  
  - `ALTER INSTANCE DISABLE INNODB REDO_LOG`  
- 활성화  
  - `ALTER INSTANCE ENABLE INNODB REDO_LOG`  
- 상태 확인  
  - `SHOW GLOBAL STATUS LIKE 'Innodb_redo_log_enabled'`  

## 4.2.12 어댑티브 해시 인덱스(Adaptive Hash Index)  
InnoDB는 자주 조회되는 B-Tree 페이지에 동적 해시 인덱스를 생성해 검색 속도를 높인다.

- 동작 방식  
  - 버퍼 풀 내 자주 액세스된 페이지 키를 해시 테이블에 저장  
  - 동등 조건 검색·IN 연산 등에서 Hash Lookup 사용  

- 파티셔닝  
  - `innodb_adaptive_hash_index_parts` (기본 8)로 내부 잠금 경합 완화  

- 장단점  
  - 장점  
    - B-Tree 탐색 비용 제거 → CPU 사용률 감소·처리량 증가  
  - 단점  
    - 디스크 I/O가 많은 환경에서는 효과 미미  
    - 메모리 오버헤드 및 히트율 관리 필요  
    - 테이블 DROP/ALTER 시 해시 인덱스 제거로 CPU 부하 증가  

- 모니터링  
  - `SHOW ENGINE INNODB STATUS\G`  – hash searches/s vs non-hash searches/s  
  - `SELECT EVENT_NAME, CURRENT_NUMBER_OF_BYTES_USED FROM performance_schema.memory_summary_global_by_event_name WHERE EVENT_NAME LIKE 'memory/innodb/adaptive hash index'`  

## 4.2.13 InnoDB vs MyISAM vs MEMORY 비교  
MySQL 8.0 기준 주요 스토리지 엔진 특징 비교  
| 특성            | InnoDB                                    | MyISAM            | MEMORY/TempTable   |
|-----------------|-------------------------------------------|-------------------|--------------------|
| 트랜잭션 지원   | MVCC·롤백·원자성 보장                     | 미지원            | 미지원             |
| 동시성          | 레코드 수준 잠금·MVCC로 높은 동시성 제공   | 테이블 수준 잠금  | 테이블 수준 잠금   |
| 외래 키         | 지원                                      | 미지원            | 미지원             |
| 복구            | 자동화된 장애 복구·Redo/Undo 로그 활용   | 체크섬·수동 복구  | 없음               |
| 메모리 처리     | 버퍼 풀 통한 캐시 및 쓰기 버퍼 제공       | 없음              | 완전 메모리 처리   |
| 변경 성능       | 해시 인덱스 오버헤드 존재                 | 빠름              | 빠르나 동시성 제한 |
