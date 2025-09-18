---
layout: single
title: "Postgrsql 프로세스 (postmaster, background, backend)"
categories: CS
tag: [DB, PostgreSQL]
search: true
typora-root-url: ../
sidebar:
  nav: "counts"









---



**[**PostgreSQL][Postgrsql 프로세스 (postmaster, background, backend) ](https://park-chanyeong.github.io){: .notice--primary}



![PostgresSQL 아키텍쳐](https://blog.kakaocdn.net/dna/md0i5/btsBbQVsUrJ/AAAAAAAAAAAAAAAAAAAAAKTctFmAmDoNHFpSXsEcYYKu6S2HQVAEqZ08P1D8WTko/img.png?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1759244399&allow_ip=&allow_referer=&signature=EKEJuFFO6xfhPDawqaFcXpFmsDo%3D)

postgreSQL은 프로세스, 시스템 운영에 필요한 파일들 , 메모리 영역으로 구성됨

주요 프로세스는 postmaster, background, backend등 크게 3가지로 구분됨.

- 프로세스들은 각자 독립된 역할을 수행하며 클라이언트 요청 처리 및 안정적 운영을 위한 여러 기능 수행

db를 구성하는 물리적 파일들은 디렉토리와 여러 파일의 형태들로 존재하고 데이터를 효율적으로 저장/관리함

메모리는 개별 프로세스를 위한 로컬 메모리와 모든 프로세스가 공유하는 공유 메모리 영역으로 나뉨

- 특히, 공유 메모리에 해당하는 shared buffer는 데이터가 처리되는 중요한 영역임
- 메모리에서 처리된 데이터는 디스크에 기록되기 전 모든 변경 사항을 WAL로그에 기록함

## 주요 프로세스

### 1. Postmaster 프로세스

Postmaster 프로세스는 인스턴스가 기동 될 때 가장 먼저 시작되며, 여러 Background 프로세스와 Backend프로세스를 생성함.

Postmaster 프로세스는 접속을 원하는 클라이언트를 기다리다가 클라이언트가 접속 시도하면 별도의 Backend 프로세스가 생성되고 이 프로세스는 클라이언트 연결이 끊기거나 강제로 종료될때까지 유지

### 2. Backend 프로세스

Postmaster 프로세스에 의해 생성된 Backend프로세스는 클라이언트와 1:1 관계를 유지하면서 클라이언트가 요청한 쿼리를 수행하고 결과를 전송함. ( Backend 프로세스 수는 max_connections 파라미터를 통해 변경 가능)

| 이름                 | 설명                                                         |
| -------------------- | ------------------------------------------------------------ |
| work_mem             | 정렬 작업 (order by, group by, distinct) 조인 작업 시 사용하는 메모리 공간 |
| maintenance_work_mem | Vaccum작업, 인덱스 생성 작업에 사용                          |
| temp_ buffers        | 세션 단위로 임시 테이블에 접근하기 위해 사용하는 메모리 공간 |
| catalog_cache        | 스키마 내부의 메타데이터를 이용할 때 사용하는 메모리 공간    |

### 3. Background 프로세스

PostgreSQL서버의 운영은 Background 프로세스에 의해 유지 되는데, 이 중 일부는 항상 백그라운드 형태로 남아있고, 일부는 작업완료되면 종료됨.

이 중 background writer와 WAL writer는 필수 프로세스며, 나머진 선택적 사용

| 이름                 | 설명                                                         |
| -------------------- | ------------------------------------------------------------ |
| Background writer    | 주기적으로 shared buffer의 dirty 페이지를 디스크에 기록      |
| Checkpointer         | 특정 간격으로 모든 dirty 페이지와 트랜잭션 로그를 디스크에 기록 |
| Autovacuum Launcher  | Autovacuum이 필요한 시점에 수행하고 관리                     |
| WAL writer           | WAL (writer-ahead log)을 디스크에 기록                       |
| Statistics collector | 세션정보 및 테이블 통계 정보와 같은 DB 사용 통계를 수집하여 pg_catalog에 정보를 반영 |
| Archiver             | WAL파일이 가득 차면 디스크에 기록                            |
| logger               | 오류 메시지를 로그 파일에 기록                               |

> Dirty 페이지는 PostgreSQL shared buffer에서 변경됐지만 아직 디스크에 기록되지 않은 데이터 페이지를 의미함. 성능 최적화를 위해 존재하며, background writer와 checkpointer가 이를 디스크에 내려주는 역할을 함.



## 메모리

![Let's get back to basics - PostgreSQL memory components](https://www.postgresql.fastware.com/hs-fs/hubfs/Images/Diagrams/img-dgm-postgresql-shared-memory.png?width=500&name=img-dgm-postgresql-shared-memory.png)

postgreSQL은 다른 db와 달리 Shared Buffer가 다소 작게 설정되도록 설계되어 있음.

메모리 크기가 1GB이상일 경우 Shared Buffer는 시스템 메모리의 25%를 사용하도록 권장함. 나머지 75%는 OS의 커널, 서비스, 캐시등에서 사용됨.

왜냐? → **공유메모리와 OS캐시를 함께 사용하여 데이터에 빠르게 접근하기 위해서!**

PostgreSQL은 Shared Buffer와 OS 캐시가 함께 동작해서 실제 물리적인 IO를 줄여 주도록 설계되어 있음.

이 외에도 디스크 컨트롤러 캐시, 디스크 드라이브 캐시 등이 존재하며 이러한 캐시들의 공통점은 모두 물리 IO를 줄여 속도를 향상시키는 데 있음.



### 1. Shared Memory (공유 메모리)

### 1-1 Shared Buffer

일반적인 db의 버퍼 캐시와 유사함.

**데이터의 변경 사항을 페이지 단위로 처리하며 물리적인 IO를 줄여 데이터에 빠르게 접근하기 위해 사용됨.**

Shared Buffer의 크기는 `shared_buffers` 파라미터로 조정 가능

(기본값 128MB, IO단위는 `block_size`에 의해 정해지고 기본값은 8KB)

### 1-2 WAL (Write-Ahead Logging)버퍼

**WAL 버퍼는 데이터들의 변경 사항들이 임시로 저장되는 영역.**

WAL 버퍼에 저장된 내용들은 시간이 지나면 영구 저장 장소인 WAL세그먼트 파일에 기록됨.

복구가 필요할 때 데이터를 재구성 하는 영역이기도 하며 `wal_buffers` 파라미터에 의해 크기 정의

### 1-3 Clog (Commit Log) 버퍼

**Clog는 트랜잭션의 커밋 상태를 기록하는 로그 파일임.**

이 파일에는 **In_progress, Commited, Aborted** 와 같은 정보들이 저장됨.

Clog 버퍼를 설정할 수 있는 파라미터는 없으며, DB엔진에 의해서 자동으로 관리됨.

### 1-3 Lock Space

**Shared Buffer에서 락 관련 내용을 보관하는 영역으로 모든 유형의 락 정보를 저장함.**

이곳에 저장된 락 정보는 트랜잭션들의 순차성을 보장하여 데이터들의 일관성과 무결성을 유지함.

메모리에서 저장할 수 있는 락 개수는 다음과 같이 계산됨.

```html
max_locks_per_transaction * (max_connection + max_prepared_transactions)
```

### 1-4 Other Buffers

이 영역은  통계정보, Two Phase Commit, Checkpoint 및 Auto Vacuum 등 다양한 백그라운드 프로세스를 위한 메모리 영역임.

## 2 Local Memory (로컬 메모리)

로컬 메모리는 서버 인스턴스 내에서 Backend 프로세스가 쿼리를 수행하기 위해 사용되는 영역.

Backend 프로세스는 사용자가 요청한 쿼리를 실행 후 결과를 전송하는 임무를 수행함.

로컬 메모리는 개별 세션에 대한 영역을 의미하며 로컬 메모리의 전체 크기는 사용자 접속 수의 최댓값을 고려해서 설정하는 것이 좋다~ 기본적으로 세션 단위로 할당되지만, 트랜잭션 단위의 크기로도 설정이 가능

- 세션과 트랜잭션 단위 메모리 설정

```sql
--- 세션 단위
SET work_mem = '16MB';

--- 트랜잭션 단위
SET LOCAL work_mem = '16MB';

---초기값으로 복원?
RESET work_mem;
```

로컬 메모리의 주요 파라미터는

| 이름                                                         | 설명                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| maintenace_work_mem                                          | 다양한 작업에 사용되는 메모리 공간으로 default는 64MB, work_mem의 1.5배를 권장함. |
| Vacuum 관련 작업, 인덱스 생성, 테이블 변경, 외래 키 추가 등 이와 같은 작업에 사용 |                                                              |
| temp_buffers                                                 | Temp 테이블 사용을 위한 공간이며 세션단위로 할당됨 (default 8MB) |
| work_mem                                                     | 쿼리의 정렬 작업이나 해시 연산 수행 시 사용되는 메모리 공간으로 default 4MB |
| shared_buffers / max_connections 값으로 계산해서 적용할 것을 권장 |                                                              |
| catalog_cache                                                | 시스템 카탈로그 메타 데이터를 사용할 때 필요한 공간.         |
| 많은 세션들이 메타데이터를 조회하면 디스크 IO 땜시 성능 저하 가능성있음 |                                                              |
| 각 세션마다 Catalog Cache를 사용하면 디스크  IO  성능을 줄일 수 있음 |                                                              |
| Optimizer / Executor                                         | 수행한 쿼리들에 대한 최적의 실행 계획 수립 및 실행 시 사용되는 메모리 공간 |