# 4.4 MySQL 로그 파일  
- 로그 파일은 서버 상태 진단의 첫 단추
- 에러·쿼리·슬로우 로그를 통해 문제 원인 파악 가능  

## 4.4.1 에러 로그 파일  
MySQL 실행 중 발생하는 에러·경고 메시지를 기록한다.

- 위치  
  - `my.cnf`의 `log_error` 경로  
  - 미설정 시 데이터 디렉터리에 `.err` 확장자로 생성  

- 주요 메시지 유형  
  - **시작 과정 정보 및 에러**  
    - 설정 파라미터 적용 여부 확인  
    - ‘ready for connections’ 메시지로 정상 기동 판단  
  - **비정상 종료 후 InnoDB 트랜잭션 복구**  
    - 미완료 트랜잭션 정리 및 리두·언두 로그 재처리  
    - `innodb_force_recovery` 단계별 복구 필요 시 활용  
  - **쿼리 처리 도중 에러·경고**  
    - 실행 중 예기치 못한 오류나 복제 경고 기록  
  - **Aborted connection**  
    - 비정상 커넥션 종료 시 클라이언트 정보 기록  
    - `max_connect_errors` 초과 시 호스트 차단 알림  
  - **SHOW ENGINE INNODB STATUS 결과**  
    - 모니터링 명령이 출력하는 상태 정보 대량 기록  
    - 사용 후 비활성화 권장  
  - **MySQL 종료 메시지**  
    - 정상 종료: ‘Received SHUTDOWN from user…’  
    - 비정상 종료: 세그멘테이션 폴트 스택 트레이스  

## 4.4.2 제너럴 쿼리 로그 파일  
모든 쿼리 요청을 기록해 실행 패턴 분석에 사용한다.

- 활성화  
  - `SET GLOBAL general_log = ON`  

- 저장 위치  
  - `SHOW GLOBAL VARIABLES LIKE 'general_log_file'`  

- 출력 대상  
  | log_output | 저장 대상                          |
  |------------|------------------------------------|
  | FILE       | 지정된 파일 경로에 기록           |
  | TABLE      | `mysql.general_log` 테이블에 기록 |

## 4.4.3 슬로우 쿼리 로그  
`long_query_time` 이상 소요된 쿼리를 기록해 병목 식별에 활용한다.

- 기록 조건  
  - 쿼리 정상 완료 시 `Query_time` ≥ `long_query_time`  

- 저장 대상  
  - `log_output` 설정에 따라 FILE 또는 TABLE  

- 기록 항목  
  - `# Time`: 쿼리 종료 시점  
  - `# User@Host`: 클라이언트 계정 및 호스트  
  - `# Query_time`, `Lock_time`  
  - `# Rows_sent`, `Rows_examined`  

- 분석 도구 예시  
  - Percona Toolkit `pt-query-digest`로 통계·랭킹·상세 정보 생성  

- 슬로우 로그 구조  
  - **통계 요약**  
    - 전체 쿼리의 Exec time·Lock time 평균·최댓값 등  
  - **쿼리 랭킹**  
    - Response time 기준 상위 쿼리 목록 및 실행 횟수  
  - **상세 리포트**  
    - 각 Query ID별 히스토그램, 테이블 구조, 실행 계획 등  
