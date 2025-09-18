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