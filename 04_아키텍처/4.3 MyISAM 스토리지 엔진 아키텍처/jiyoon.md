# 4.3 MyISAM 스토리지 엔진 아키텍처  
MySQL 8.0 환경에서는 InnoDB 기반으로 대부분 재편됐지만, MyISAM 아키텍처를 이해하면 키 캐시와 OS 캐시 역할 차이를 알 수 있다.

## 4.3.1 키 캐시  
- MyISAM의 인덱스 전용 버퍼링 메커니즘  
- 역할  
  - 인덱스 블록을 메모리에 유지해 디스크 I/O 최소화  
  - 데이터 파일 자체는 버퍼링하지 않음  
- 히트율 계산  
  - Hit_rate = 100 − (Key_reads / Key_read_requests × 100)  
  - Key_read_requests: 캐시로부터 인덱스 블록 요청 횟수  
  - Key_reads: 디스크에서 실제 읽은 횟수  
- 설정 및 제한  
  - `key_buffer_size`로 기본 키 캐시 크기 지정  
  - 32bit OS: 최대 4 GB 설정 제한  
  - 64bit OS: `key_buffer_size` 외에 명명된 키 캐시 추가 가능  
- 명명된 키 캐시 사용 예  
  ```sql
  CACHE INDEX db1.board, db2.board IN kbuf_board
  CACHE INDEX db1.comment, db2.comment IN kbuf_comment
  ```

## 4.3.2 운영체제의 캐시 및 버퍼  
MyISAM 데이터 파일 I/O는 OS 레벨 캐시에 의존한다.

- 특징  
  - OS 파일 시스템 캐시가 남은 메모리를 사용해 디스크 I/O 가속  
  - 애플리케이션이 메모리 과다 사용 시 OS 캐시 공간 부족으로 성능 저하  

- 권장 사항  
  - MyISAM 테이블 주요 사용 시 전체 메모리 중 최대 40 % 이하로 `key_buffer_size` 설정  
  - 나머지 메모리를 OS 캐시에 할당해 데이터 파일 캐싱 지원  

## 4.3.3 데이터 파일과 ROWID 구조  
MyISAM은 데이터 파일을 Heap처럼 사용하며, 물리 주소는 ROWID로 관리된다.

- 저장 순서  
  - INSERT 순서대로 데이터 파일에 기록됨 (클러스터링 없음)  

- ROWID 유형  
  - **고정 길이 ROWID**  
    - `MAX_ROWS` 옵션 지정 시 사용  
    - 4바이트 정수로 INSERT 순번 저장  
  - **가변 길이 ROWID**  
    - `myisam_data_pointer_size`(기본 7) 바이트 사용  
    - 첫 바이트: ROWID 길이, 나머지: 실제 오프셋  
    - 최대 데이터 파일 크기 = 2^(8×(pointer_size−1)) − 1  
