---
layout: single
title: "PostgreSQL Logical Structure 논리적 구조"
categories: CS
tag: [DB, PostgreSQL]
search: true
typora-root-url: ../
sidebar:
  nav: "counts"









---



**[PostgreSQL][PostgreSQL Logical Structure 논리적 구조](https://park-chanyeong.github.io)**
{: .notice--primary}



## PostgreSQL 논리구조

## 

Postgresql 의 구조는 **논리 구조**와 **물리구조**로 나뉜다. 논리 구조는 오브젝트, 스키마, 데이터베이스, 롤, 테이블 스페이스가 해당된다.

1. **튜플** : 페이지 단위로 저장되며, 하나의 페이지에 여러개의 튜플이 존재
2. **오브젝트** : 페이지의 집합
3. **스키마** : 오브젝트의 집합
4. **데이터베이스** : 스키마의 집합
5. **클러스터** : 데이터베이스와 Role 그리고 테이블 스페이스의 집합

![DB 인사이드 | PostgreSQL Architecture - 3. Logical Structure](https://blog.kakaocdn.net/dna/DUR5h/btrAtKPSmnZ/AAAAAAAAAAAAAAAAAAAAAIoN1fpClek2BvKTcOs8UzGtKe0nJEkz8h74MCAXy1XM/img.jpg?credential=yqXZFxpELC7KVnFOS48ylbz2pIh7yKj8&expires=1759244399&allow_ip=&allow_referer=&signature=ERCVay86tlcYvKMPBNn20SDNLQU%3D)

### 1. Cluster

**클러스터란 논리적인 개념이며 하나 이상의 db와 Role, tablespace의 집합을 의미함.**

하나의 인스턴스는 하나의 클러스터만을 관리하며 한번에 여러 DB에 서비스를 제공할 수 있음,

![데이터베이스 클러스터의 논리적인 구조](https://coiy.github.io/assets/pg_db_cluster.png)

### 2. Database

**Postgresql에서의 클러스터는 여러개의 Database로 구성할 수 있음.**

최초 initdb()를 수행하게 되면 **Template0**, **Template1**, **Postgres** 3개의 DB가 생성됨!

템플릿 DB는 이름부터 보이듯이 DB 생성 시 템플릿을 제공하기 위한 DB로,

사용자가 Template1에 객체를 추가한 후에 새로운 DB를 생성하여 사용하면 Template1에서 생성한 내용들이 복제됨. 따라서 추가작업 안하고도 템플릿에 따라 오브젝트들을 새롭게 생성한 DB에도 동일하게 사용 가능!

근데 왜 Template DB가 2개냐면, Template0은 Template1의 초기상태와 동일하며 postgresql에서 제공하는 표준 객체만 포함되어 있고 수정이나 변경 불가능!

그래서 Template1에 여러 템플릿을 추가해서 사용하다가 초기 상태로 복원이 필요할 때 Template0을 복제해서 template1을 생성하면 됨.

사용자가 Template 옵션 없이 DB를 생성하게 되면 기본적으로 Template1 DB를 복제하여 생성됨.

```sql
--- template0 을 복제해서 생성하려면 다음과 같은 명령어
CREATE DATABASE pcy_example_db TEMPLATE template0;
```

![PostgreSQL之template0、template1和postgres库- 墨天轮](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcQHHOq7MjcfXUE9px37gUlkeB0FQwCtDLquvg&s)

여기서

`datistemplate` 은 DB생성을 위한 템플릿용 여부 표시 (t면 create권한 있는 사용자는 복제할 수 있고 f면 슈퍼 유저와 DB 소유자 만이 복제할 수 있는 권한이 있음.)

`datallowconn` 은 DB에 접속여부를 표시. 보면 template0은 f이기 때문에 새로운 접속이 허용되지 않아 수정도 불가능하지만 template1은 접속,수정 모두 가능

### 3. Schema

**스키마는 다양한 오브젝트들의 집합을 의미하며 하나의 DB는 다수의 스키마를 소유할 수 있음.**

테이블 생성 시 스키마를 지정하지 않으면 Public스키마로 생성되는데 Create 권한을 포함은 모든 권한이 부여된 기본 스키마임.

(동일 DB라도 스키마 명이 다르면 동일 이름의 테이블이 존재할 수 있기떄문에 가능한 동일 이름의 테이블을 사용하지 않기로 하자)

`pg_catalog.pg_namespace` 를 조회하면 현재 DB에 존재하는 스키마를 확인할 수 있다

```sql
SELECT * FROM pg_catalog.pg_namespace
```

해당 명령어를 하면

![image-20250919034735860](/images/2025-09-16-postgresql1/image-20250919034735860.png)

다음과 같이 나오는디,

- **`pg_toast`** : TOAST와 관련된 오브젝트들을 위해 사용됨.
- **`pg_catalog`** : 시스템 카탈로그 정보를 의미하며 테이블, 컬럼, 메타데이터, 파라미터, 락등의 내부 설정 및 오퍼레이션 정보를 확인 가능
- **`public`** : 사용자 계정의 기본 스키마로 스키마를 설정하지 않으면 해당 스키마로 설정됨.
- **`information_schema`** : 현재 접속한 DB의 오브젝트에 대한 정보를 담고 있는 뷰들로 이루어짐

### 4. Role

role이란 **DB 관련 권한들에 대한 특정 이름들의 집합**으로 user와 기능적인 차이는 없음.

role에 로그인 권한을 부여하면 일반 사용자 계정과 동일하게 사용할 수 있으며, 개별 DB가 아닌 클러스터 레벨에서 공통으로 사용됨.

```sql
User = Role + Login Permission
```

Role 권한의 종류는 다음과 같음,.

| **Role type**        | 설명                                     |
| -------------------- | ---------------------------------------- |
| **SUPERUSER**        | LOGIN Role을 제외한 모든 Role을 포함함.  |
| **LOGIN**            | Database에 접속하기 위한 Role            |
| **CREATEDB**         | DB를 생성하기 위한 Role                  |
| **CREATEROLE**       | Role을 생성,수정,삭제,변경하기 위한 Role |
| **REPLCATION**       | Replication을 사용하기 위한 Role         |
| **PASSWORD**         | 패스워드 설정                            |
| **CONNECTION LIMIT** | 커넥션 개수 설정                         |
| **INHERIT**          | Role에 할당한 권한들을 상속              |
| **BYPASSRLS**        | row security system을 우회하는 Role      |

### 5. Object

오브젝트는 데이터를 저장하거나 참조하는데 사용되는 데이터 구조이며 테이블,인덱스,뷰,시퀀스,프로시저 등이 포함됨.

| 오브젝트      | 설명                                                         |
| ------------- | ------------------------------------------------------------ |
| **Table**     | 데이터를 저장하는 기본단위, 열과 행으로 구성됨. 각 열은 특정 유형 데이터를 저장하고, 각 행은 테이블의 한 레코드(튜플)를 나타냄 |
| **View**      | 하나 이상의 테이블에서 데이터를 선택적으로 보여주는 가상 테이블. 뷰는 DB내의 데이터를 요약하거나, 복잡한 쿼리를  단순화 하는 데 사용함 |
| **Index**     | 데이터 검색 속도를 향상시키기 위해 사용되는 오브젝트. 인덱스는 테이블 하나 or 여러개의 열을 기준으로 생성될 수 있으며, <br />데이터 검색 시 빠른 데이터 접근을 가능하게 함! |
| **Sequence**  | 일련번호 생성기로 사용되며, 주로 자동 증가 필드(ex. 기본키)에 사용됨! 시퀀스를 통해 고유한 숫자 값을 차례대로 생성할 수 있음 |
| **Function**  | 특정 작업을 실행하는 SQL문이나 코드 블록. 함수는 데이터 처리, 계산 수행, 데이터 변환등 작업에 사용 |
| **Procedure** | 프로시저는 DB에서 실행할 수 있는 저장된 명령어 집합을 정의하여 데이터 처리를 자동화하고 효율성을 개선하는 역할을 함 |
| **Trigger**   | 특정 이벤트 ( insert, delete, update)가 발생할 때 자동으로 실행되는 함수로, 데이터 무결성을 유지하거나 특정 조건에 따라 자동으로 작업을 수행하는 데 사용됨. |




### 6. Tablespace

**테이블 스페이스는 오브젝트가 파일 형태로 저장되는 위치를 나타내며 일반적으로 데이터 저장 공간을 관리하는 목적으로 사용됨!**

Postgresql 에서는 하나의 인스턴스에서 여러개의 DB를 구성할 수 있고, 여러 DB가 동일한 테이블 스페이스를 사용할 수 도 있음.

특히, 테이블 스페이스를 통해 물리적인 디스크 저장 공간을 제어할 수 있음.

(ex. 빈번하게 사용되는 오브젝트는 SSD같이 고성능 디스크 영역에 테이블 스페이스를 구성하고, 아닌것들은 별도 디스크영역에 테이블 스페이스를 구성하여 관리할 수 있음)

Postgresql은 두 개의 기본 테이블스페이스와 사용자 정의 테이블 스페이스가 존재하는디

```sql
SELECT * FROM pg_tablespace;
```

![image-20250919040323412](/images/2025-09-16-postgresql1/image-20250919040323412.png)

- **`pg_default`** : 모든 DB에서 사용되는 기본 테이블 스페이스를 의미하며 모든 사용자 데이터를 저장.
- **`pg_global`** : 물리적 저장 위치는 $PGDATA/global 이며 DB 클러스터 레벨에서 관리되는 테이블들을 저장함.
  - `pg_authid, pg_database, pg_shdepend, pg_shdescription, pg_auth_members, pg_db_role_setting, pg_tablespace` 와 같이 클러스터 레벨에서 관리되는 테이블들이 저장되며 모든 DB에서 동일한 정보를 제공함.
- 사용자 정의 테이블스페이스는 DB생성시 Owner와 Location을 지정할 수 있으며 저장되는 물리적인 위치는 $PGDATA/pg_tblspc임. 해당 디렉토리를 조회하면 Location위치를 가리키는 심볼릭 링크로 연결된 것을 확인 가능

> 테이블스페이스 생성 시 데이터 디렉토리는 실제 데이터가 저장된 위치를 나타내며, 파일 시스템 권한을 가진 사용자가 생성할 수 있음. 테이블 스페이스를 사용하는 모든 오브젝트는 데이터 디렉토리에 파일 형태로 저장됨. 테이블스페이스가 장애나면 DB클러스터는 동작하지 않으므로, 데이터가 저장되는 테이블스페이스는 임시 디렉토리에 생성하지 않도록 조심하자..

### 7. Relation

RDB에서 Relation이란 표현은 여러가진디 **테이블이나 인덱스와 같이 데이터를 포함하는 DB개체**를 말함.

인덱스는 테이블과 종속적인 관계가 있지만 뷰(쿼리로 구성된 개체)나 시퀀스는 다른 개체들과 관계가 있지 않을 수 있긴 한데 postgresql에서는 이러한 모든 개체를 릴레이션이라 합니더

### 8. Page

**페이지란 디스크IO가 발생하는 최소 단위이며 고정 길이의 데이터 블록을 의미함.**

기본 페이지 크기는 8KB~32KB까지 정의 가능하지만 **일반적으로는 기본 페이지 크기인 8KB를 사용**

모든 테이블과 인덱스는 페이지 배열로 이뤄져 있으며 각각의 페이지는 헤더 영역과 데이터 영역으로 나눠짐.

(한번 정의된 페이지 크기는 변경x)

### 9. Tuple

튜플은 테이블의 Record나 row가 물리적으로 저장된 형태이며 데이터를 저장하는 최소 단위.

튜플의 헤더에는 트랜잭션ID, 가시성 맵(Visible Map)등 다양한 메타데이터가 존재함.

튜플은 MVCC 적용에 따라 Live튜플과 Dead튜플이 존재할 수 있음.

Live튜플은 데이터 처리 시 현재 버전의 튜플을 의미하고, Dead튜플은 DML에 의해 변경된 이전 버전의 튜플을 의미함.

따라서, Dead튜플은 참조하는 트랜잭션이 없으면 물리적으로는 존재하지만 삭제되어야 할 과거 이미지를 의미하므로 Vacuum에 의해 제거될 수 있다~~

### 10. Extension

익스텐션은 DB의 특정 기능을 확장하기 위한 모듈로 DB에서 제공되지 않는 기능을 사용자가 설치해서 기존 DB 기능처럼 사용하는 것.

(ex. 새로운 함수 추가, 대용량 데이터 처리, 외부 프로그램)

Create, Drop, Alter Extension명령어로 설치, 삭제, 변경할 수 있으며, 바이너리 배포판에 기본적으로 포함되는 익스텐션은 공식 메뉴얼을 통해 확인 가능하긴 함

```sql
--- 익스텐션 설치,삭제
CREATE EXTENSION 익스텐션이름
DROP EXTENSION 익스텐션이름

---기본 포함된 익스텐션 확인
SELECT * FROM pg_available_extensions;
```

![image-20250919042530207](/images/2025-09-16-postgresql1/image-20250919042530207.png)