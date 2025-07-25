# 1회차 내용 정리
범위 : Ch.4 아키텍처 – 4.1 MySQL 엔진 아키텍처 (Ch.1‑3은 사전 읽기)
## 4.1 MySQL 엔진 아키텍처

### 1. MySQL 서버 구성

- MySQL 엔진: 쿼리 파서, 옵티마이저, 실행 엔진 등 포함
- 스토리지 엔진: 실제 데이터 저장/조회 담당 (예: InnoDB, MyISAM 등)
    ```
    # 스토리지 엔진 지정 예시
    CREATE TABLE test_table (fd1 INT, fd2 INT) ENGINE=InnoDB;
    ```
- 핸들러 API: MySQL 엔진과 스토리지 엔진 간 인터페이스
  - 핸들러 요청이란?: MySQL 엔진의 쿼리 실행기에서 데이터를 쓰거나 읽어야 할 때, 각 스토리지 엔진에 쓰기 또는 읽기를 요청하는 것
    ```
    # 핸들러 상태 변수 조회 예시
    SHOW GLOBAL STATUS LIKE 'Handler%';
    ```

### 2. MySQL 스레드 구조

- 스레드 기반 구조: 포그라운드(클라이언트 요청), 백그라운드(내부 작업) 스레드로 구성
```
# MySQL 서버에서 실행 중인 스레드 목록 확인 방법
SELECT thread_id, name, type, processlist_user, processlist_host
FROM performance_schema,.threads ORDER BY type, thread_id;
```
- 포그라운드 스레드: 각 클라이언트 사용자가 요청하는 쿼리 문장 처리, 커넥션별 1:1 스레드 생성, 쿼리 처리 후 캐시로 복귀
- 백그라운드 스레드:
  - Insert Buffer 병합
  - 로그 기록
  - 버퍼 풀 플러시
  - 읽기 스레드
  - 데드락 모니터링 등

### 3. 메모리 구조

- 글로벌 메모리 영역: 테이블 캐시, InnoDB 버퍼 풀 등
- 로컬 메모리 영역: 커넥션 버퍼, 정렬 버퍼 등
  - 필요 시 할당, 종료 시 해제
  - 각 커넥션이 독립적으로 사용

### 4. 플러그인 & 컴포넌트 아키텍처

- 플러그인 구조: 스토리지 엔진 외에도 인증, 검색 파서 등 다양한 기능 확장
- 컴포넌트 구조 (MySQL 8.0+):
  - 플러그인의 단점을 보완
  - 캡슐화, 컴포넌트 간 통신 가능
  - 예시: validate_password 컴포넌트 설치
    ```
    INSTALL COMPONENT 'file://component_validate_password';
    SELECT * FROM mysql.component;
    ```
### 5. 쿼리 실행 구조

1. 파서: 쿼리를 토큰 단위로 분해
2. 전처리기: 개체 존재 여부 및 권한 확인
3. 옵티마이저: 최적의 실행 계획 수립
4. 실행 엔진: 실행 계획에 따라 핸들러 호출
5. 핸들러: 스토리지 엔진에서 실제 데이터 처리

### 6. 기타 아키텍처 요소

#### 쿼리 캐시 (제거됨)
- MySQL 8.0에서 완전히 제거됨
- 이유: 동시 처리 성능 저하, 많은 버그 원인

#### 스레드 풀
- Percona 서버 또는 엔터프라이즈 버전에서 사용 가능
- 제한된 스레드 수로 요청 처리 → 컨텍스트 스위치 감소, CPU 사용 효율화

### 7. 트랜잭션 기반 메타데이터

- 기존: FRM, TRG, PAR 등 파일 기반 → 비정상 종료 시 손상 위험
- MySQL 8.0: InnoDB 기반 시스템 테이블에 저장
- SDI 파일: 비 InnoDB 테이블용 메타정보 저장 (CSV, MyISAM 등)

### 8. InnoDB 아키텍처

- 레코드 기반 잠금 지원
- 높은 동시성, 안정성, 성능
- 상세 구조: 버퍼 풀, 리두 로그, 언두 로그 등 (그림 4.9 참조)
