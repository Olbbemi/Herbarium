---
kind: concept
id: embedded-db
title: 임베디드 데이터베이스
summary: |
  임베디드 DB 는 서버 없이 라이브러리 형태로 앱 프로세스 안에서 실행되는 데이터베이스다.
  SQLite(OLTP)와 DuckDB(OLAP)가 대표적이며, 저장 구조와 실행 모델이 근본적으로 달라 적합한 워크로드가 전혀 다르다.
category: database
tags:
  - embedded-db
  - sqlite
  - duckdb
  - oltp
  - olap
  - columnar-storage
  - vectorized-execution
relations: [contrasts_with:mysql, contrasts_with:postgresql, same_family:sqlite, same_family:duckdb, same_family:leveldb, same_family:rocksdb, same_family:berkeley-db, related:wal, related:oltp, related:olap, related:columnar-storage, related:vectorized-execution, related:file-lock]
pending:
  - duckdb: DuckDB 세부 동작(단건 insert 지연 원인, 벡터 크기 설정 변경 여부) -- knowledge/duckdb.md 생성 후 이동
---

## 임베디드 DB 란

### 클라이언트-서버 DB 와의 차이

MySQL, PostgreSQL 은 별도 서버 프로세스가 항상 실행 중이고, 앱은 네트워크(소켓)로 연결해 쿼리를 보낸다.
임베디드 DB 는 서버 프로세스가 없다. 라이브러리 형태로 앱 프로세스 안에서 직접 실행되며, 쿼리는 함수 호출로 처리된다.

```
클라이언트-서버 DB:
  앱 프로세스 --[네트워크/소켓]--> DB 서버 프로세스 --[파일 I/O]--> 저장소

임베디드 DB:
  앱 프로세스 --[함수 호출]--> DB 엔진(라이브러리) --[파일 I/O]--> 저장소
```

앱 프로세스와 DB 엔진이 같은 주소 공간 안에 있다는 점이 핵심이다. "DB 에 접속한다"는 표현은 엄밀히는 성립하지 않으며, 파일을 여는 것에 가깝다.

### 설계 철학

임베디드 DB 의 핵심 설계 원칙은 단순성과 이식성이다.

- 제로 설치: 라이브러리 하나를 링크하면 끝. 데몬 설치, 서비스 등록, 포트 개방 없음
- 제로 관리: DBA 없이 운영 가능. 튜닝·모니터링·백업이 OS 수준에서 가능
- 이식성: DB 전체가 파일 하나. 복사로 이동·백업·버전 관리 가능
- 결정론적: 외부 서비스 상태에 의존하지 않아 테스트·재현이 쉬움

이 철학은 "애플리케이션과 DB 가 일체"라는 개념에서 나온다. DB 를 별도로 프로비저닝하는 게 아니라, 앱을 배포하면 DB 도 함께 딸려오는 구조다.

## 공통 특징

- 서버 설치·관리 불필요 -- 라이브러리 링크만으로 동작
- 네트워크 레이어 없음 -- 함수 호출 직접 접근, 레이턴시 거의 0
- 단일 파일(또는 인메모리) -- DB 전체가 파일 하나, 복사가 곧 백업·이동
- 제로 설정 -- 서버 설정, 포트, 계정 관리 없음
- 낮은 리소스 -- 별도 프로세스가 없어 메모리·CPU 점유 최소
- 앱 프로세스 라이프사이클 -- DB 는 앱이 살아 있는 동안만 동작

## 공통 한계

- 다중 프로세스 동시 쓰기에 제약 -- 여러 앱이 같은 DB 에 동시에 쓰는 구조에는 부적합
- 수평 확장 불가 -- 서버가 없으니 클러스터링·레플리케이션 없음
- 네트워크 접근 불가 -- 자체 네트워크 프로토콜이 없어 원격 머신에서 연결할 수 없음
  (libSQL, MotherDuck 같은 써드파티 래퍼가 있으나 임베디드 이점이 희석됨)

## 저장 모델 유형

임베디드 DB 는 하나의 카테고리가 아니라 다양한 저장 모델로 나뉜다.

| 유형 | 저장 방식 | 주요 구현체 | 최적 워크로드 |
|------|-----------|-------------|---------------|
| 행(row) 기반 관계형 | 한 레코드의 모든 컬럼이 붙어서 저장 | SQLite | OLTP -- 단건 CRUD |
| 열(column) 기반 관계형 | 같은 컬럼 값들이 연속 저장 | DuckDB | OLAP -- 대규모 집계 |
| 키-값(key-value) | 키 -> 값 매핑, LSM-Tree 또는 B-Tree | LevelDB, RocksDB | 고빈도 쓰기, 순차 스캔 |
| 문서(document) | 구조화되지 않은 JSON/BSON 저장 | SQLite JSON1, 각종 임베디드 문서 DB | 스키마 유동적인 데이터 |

같은 "임베디드 DB" 이름 아래라도 저장 방식과 쿼리 엔진이 근본적으로 달라 워크로드에 따라 적합한 선택이 전혀 다르다.

## 내부 동작 원리

### WAL (Write-Ahead Log)

SQLite, DuckDB 를 포함한 대부분의 임베디드 DB 가 내부적으로 WAL 을 사용한다. 목적과 활용 범위는 구현체마다 다르다.

핵심 원리: 데이터를 원본 파일에 직접 덮어쓰는 대신, 변경 내용을 별도 로그 파일에 먼저 순차적으로 기록한다. 실제 데이터 파일 반영(체크포인트)은 나중에 한다.

```
WAL:
  쓰기 요청 -> WAL 파일에 append -> 즉시 완료
  체크포인트 -> WAL 내용을 데이터 파일에 병합 (주기적 또는 명시적)
```

공통 장점:

- 크래시 복구: 로그를 재생하면 마지막 커밋 시점으로 복구 가능
- append-only 쓰기로 디스크 I/O 패턴이 순차적

구현체마다 다른 부분:

- 동시 읽기-쓰기 허용 여부 -- WAL 을 동시성 메커니즘으로도 쓰는지, 크래시 복구 전용인지
- 체크포인트 시점과 방식
- WAL 파일의 외부 가시성 (보조 파일이 생기는지 여부)

세부 동작은 각 구현체 문서를 참고한다.

### 파일 락 -- 동시성 제어

임베디드 DB 에는 별도 서버가 없어 OS 레벨의 파일 락으로 동시 접근을 제어한다.

공통 원칙:

- 읽기는 공유 가능 -- 여러 프로세스가 동시에 읽기 가능
- 쓰기는 배타적 -- 쓰기 시점에는 다른 쓰기를 차단
- OS 파일 락 기반이므로 앱에서 별도 뮤텍스를 구현할 필요가 없음

구현체마다 다른 부분:

- 락 단계의 세분화 정도 (단순 배타 락 하나 vs 다단계 프로토콜)
- 읽기와 쓰기의 동시성 허용 범위
- 락 경합 시 대기 방식 (즉시 에러 vs 타임아웃 대기)

주의: 네트워크 파일시스템(NFS 등)에서는 OS 파일 락이 신뢰할 수 없다. 대부분의 임베디드 DB 구현체가 공식적으로 경고한다.

## 언제 임베디드가 맞는가

| 상황 | 선택 |
|------|------|
| 단일 프로세스, 단일 머신 | 임베디드 DB |
| 다중 프로세스, 단일 머신, 쓰기 경합 낮음 | SQLite (WAL 모드) 까지 가능 |
| 다중 머신, 또는 높은 동시 쓰기 | 클라이언트-서버 DB |
| 분석 쿼리, 단일 프로세스 | DuckDB |
| 분석 쿼리, 여러 서비스가 공유 | 클라이언트-서버 OLAP |

공유가 필요해지는 순간 임베디드의 이점이 사라지고 클라이언트-서버가 맞는 선택이 된다.

---

## 대표 구현

### SQLite -- 임베디드 OLTP

- 행(row) 기반 저장, 단건 CRUD 에 최적화
- ACID 트랜잭션, B-Tree 인덱스
- WAL 모드로 writer 1명 + 다수 reader 동시 접근 가능
- 다중 프로세스 쓰기: WAL + busy_timeout 설정으로 파일 락 기반 직렬화

> **OLTP(Online Transaction Processing)**
> - 실시간으로 발생하는 소규모 트랜잭션을 빠르게 처리하는 워크로드.
> - 단건 읽기·쓰기·수정·삭제가 높은 빈도로 발생하며, 한 트랜잭션이 밀리초 단위로 끝나야 한다.

### DuckDB -- 임베디드 OLAP

- 열(column) 기반 저장, 대용량 집계·스캔에 최적화
- 벡터화 실행(vectorized execution): 2,048개 값을 묶어 SIMD 연산
- 기본적으로 단일 프로세스 독점 쓰기. 읽기 전용으로는 다중 프로세스 동시 접근 가능
- 다중 프로세스 쓰기가 필요하면 단일 writer 프로세스 + IPC 패턴 필요

> **OLAP(Online Analytical Processing)**
> - 대량의 데이터를 집계·분석하는 워크로드.
> - 수백만~수억 행을 스캔해 합계·평균·그룹핑 같은 통계를 뽑아내는 쿼리 위주이며, 트랜잭션 빈도는 낮고 쿼리 하나당 처리량이 크다.

---

## SQLite vs DuckDB 비교

| 특성 | SQLite | DuckDB |
|------|--------|--------|
| 저장 단위 | 행(row) | 열(column) |
| 실행 모델 | 행 단위 인터프리터 | 벡터화 / 병렬 |
| 최적 워크로드 | OLTP -- 단건 조회·삽입·수정 | OLAP -- 대량 스캔·집계 |
| 트랜잭션 빈도 | 높은 빈도의 소규모 트랜잭션 | 낮은 빈도의 대규모 쿼리 |
| 다중 프로세스 쓰기 | WAL + busy_timeout 으로 직렬화 | 기본 불가, 전담 writer 프로세스 필요 |
| 다중 프로세스 읽기 | 항상 가능 | 읽기 전용 모드에서 가능 |
| bulk read 성능 | 느림 (불필요 컬럼도 I/O) | 빠름 (필요 컬럼만 I/O + 압축) |
| 단건 insert 성능 | 빠름 | 느림 |

분석 벤치마크에서 DuckDB 가 SQLite 보다 30~50배 이상 빠른 경우가 있고, 반대로 고빈도 소규모 insert/update 에서는 SQLite 가 10~500배 빠르다.
워크로드 방향이 반대여서 하나의 엔진이 둘 다 최적일 수 없다.

---

## 흔한 오해

> **오해:** "둘 다 파일 하나짜리 DB 니까 비슷하다"
> - 인터페이스(단일 파일, SQL, 서버 불필요)는 비슷하지만 저장 포맷과 실행 엔진이 근본적으로 다르다.
> - 같은 SQL 을 쓴다고 같은 성능이 나오지 않는다.

> **오해:** "DuckDB 가 더 빠르니까 SQLite 를 대체하면 된다"
> - DuckDB 는 단건 insert/update 가 많은 OLTP 에서 SQLite 보다 훨씬 느리다.
> - 특히 모바일·임베디드처럼 쓰기가 빈번하고 동시 접근이 있는 환경에서는 SQLite 가 맞다.

> **오해:** "bulk read 는 인덱스를 잘 만들면 SQLite 도 빠르다"
> - 인덱스는 행 기반 조회에 효과적이지만 대규모 집계에서는 열 저장 + 벡터화 실행의 구조적 이점을 따라가기 어렵다.
> - 특히 컬럼 수가 많고 그 중 일부만 집계하는 쿼리일수록 격차가 커진다.

> **오해:** "임베디드 DB 를 원격 서버에 두고 쓸 수 있다"
> - 자체 네트워크 프로토콜이 없어 기본적으로 불가능하다.
> - 써드파티 래퍼(libSQL, MotherDuck 등)가 있지만 임베디드의 이점이 희석된다.
> - 원격 접근이 필요하면 처음부터 클라이언트-서버 DB 를 쓰는 게 맞다.

---

## 참고 자료

- [DuckDB: An Architectural Deep Dive into the In-Process OLAP Engine](https://thinhdanggroup.github.io/duckdb/)
- [DuckDB vs SQLite: Which Embedded Database Should You Use? -- MotherDuck](https://motherduck.com/learn/duckdb-vs-sqlite-databases/)
- [DuckDB vs SQLite: A Complete Database Comparison -- DataCamp](https://www.datacamp.com/blog/duckdb-vs-sqlite-complete-database-comparison)
- [Why DuckDB -- duckdb.org](https://duckdb.org/why_duckdb)
- [Columnar Databases: Column vs Row Storage Explained -- MotherDuck](https://motherduck.com/learn/columnar-storage-guide/)
- [DuckDB Internals: Vectorized Execution, Columnar Storage, and Query Processing -- Calmops](https://calmops.com/database/duckdb/duckdb-internals/)
