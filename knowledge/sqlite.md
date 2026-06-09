---
id: sqlite
title: SQLite — Embedded RDBMS
summary: 서버 데몬 없이 앱 프로세스 안에 라이브러리로 링크돼 단일 .db 파일을 직접 읽고 쓰는 in-process 관계형 DB. Public Domain, amalgamation 단일 파일 배포, 100% 분기 커버리지로 신뢰성이 높아 1인 로컬 도구에 운영 부담을 0에 가깝게 만든다.
category: database
tags:
  - embedded-db
  - rdbms
  - sqlite
  - serverless-db
  - acid
relations:
  contrasts_with:
    - mysql
    - postgresql
  same_family:
    - duckdb
    - leveldb
    - rocksdb
    - berkeley-db
  related:
    - repository-pattern
    - hexagonal-architecture
    - wal
    - acid-transaction
    - public-domain
    - fossil-vcs
    - amalgamation
    - prepared-statement
---

## 한 줄 정의

SQLite는 "서버가 없는" 관계형 DB다. 별도 데몬을 띄우지 않고 앱 프로세스 안에 라이브러리(`libsqlite3` + `sqlite3.h`)로 링크돼 단일 `.db` 파일을 직접 읽고 쓴다.

## 핵심

### Embedded DB의 본질 — 프로세스 분리가 없다

MySQL/PostgreSQL 같은 client-server DB는 독립 데몬(`mysqld`, `postgres`), 네트워크 포트/소켓, 자체 인증·권한 시스템, 데몬 라이프사이클을 가진다. SQLite는 이 중 어느 것도 없다. SQLite의 "DB 엔진"은 함수의 집합(C 라이브러리)이며, 앱이 그 함수를 호출하는 순간 DB 엔진이 실행된다.

- 앱 프로세스 == DB 프로세스 (분리되어 있지 않음)
- "접속"은 사실상 `fopen()`에 가깝다 (`sqlite3_open()`이 `.db` 파일을 연다)
- 인증은 OS 파일 권한이 전부 (DB 자체에 사용자 개념 없음)
- "DB를 띄운다/내린다"는 동작이 없다 — 앱이 켜져 있는 동안만 DB가 살아 있다

이 구조 때문에 "serverless RDBMS" 또는 "in-process RDBMS"라 불린다.

### 라이브러리 vs CLI 바이너리

SQLite라는 이름 아래 두 가지 산출물이 있다. 헷갈리기 쉬우니 분리한다.

| 항목 | 정체 | 역할 |
|------|------|------|
| `libsqlite3.so` / `.a` | C 라이브러리 (DB 엔진 본체) | 앱이 링크. 실제 DB 동작 전부 여기서 발생 |
| `sqlite3.h` | C 헤더 | 앱이 include. API 선언 |
| `sqlite3` (CLI 바이너리) | 사람용 shell 도구 | `.db`를 사람이 직접 들여다보거나 SQL 실행 (디버깅·점검용) |

정상 데이터 흐름에는 `sqlite3` 바이너리가 등장하지 않는다. 앱이 직접 `libsqlite3` 함수를 호출해 `.db`를 읽고 쓰면 끝이다. `sqlite3` CLI는 "지금 DB 안에 뭐가 들었지?"를 확인할 때만 쓴다.

### 배포 형태 — Amalgamation 단일 파일

공식 소스는 100개 이상의 C 파일이지만, 배포 시 모두 합쳐 `sqlite3.c`(단일 C 파일, ~8.4MB, ~238K 라인) + `sqlite3.h` 두 개로 묶는다. 이게 amalgamation이다.

- 이식성: 프로젝트에 두 파일만 복사하면 끝, 외부 의존성 없음
- 컴파일러 최적화: 단일 translation unit이라 inter-procedure inlining이 잘 먹어 5~10% 더 빠른 코드
- 빌드 단순성: makefile 한 줄로 컴파일 가능

링크 방식 선택지는 두 가지다. (1) OS가 제공하는 시스템 라이브러리(`libsqlite3-dev` 등)를 링크 — 의존성 관리 도구(vcpkg/Conan) 없이 가능. (2) amalgamation 동봉 — SQLite 버전을 프로젝트가 핀하고 싶을 때 유리. 운영 부담만 보면 1인 로컬 도구엔 (1)이 가장 가볍다.

### 라이선스와 소스 관리

- 라이선스: Public Domain. 상업적 사용·재배포·수정 모두 제약 없음, 저작권 표기 의무도 없음
- 공식 VCS: Fossil (SQLite 팀이 SQLite로 직접 만든 VCS). Git이 아니다
- Git 미러: GitHub `sqlite/sqlite`에 read-only 일방향 미러 존재
- 라이선스 판매: Public Domain이 인정되지 않는 법역을 위해 "Warranty of Title"을 유료로 판매하기도 함

### 품질 보증 — 100% 분기 커버리지

SQLite가 광범위하게 쓰이는 핵심 근거는 테스트 수준이다. TH3(Test Harness #3)라는 비공개 C 테스트 슈트가 100% 분기 커버리지 + 100% MC/DC 커버리지를 달성하며, 모든 릴리스가 TH3 검증 후 배포된다(3.6.17/2009 이후 전 릴리스 통과). 이 신뢰성 때문에 Apple(iOS/macOS), Google(Android), Adobe, Mozilla 등이 자사 제품에 SQLite를 내장했고, 결과적으로 "지구상에서 가장 많이 배포된 DB 엔진"으로 추정된다.

### 동시성 모델 — WAL과 단일 writer

기본 저널 모드는 `DELETE`(rollback journal)로, writer가 쓰는 동안 reader가 막힌다. WAL(Write-Ahead Log) 모드로 바꾸면(`PRAGMA journal_mode=WAL;`):

- 여러 reader가 동시에 읽기 가능
- writer는 여전히 한 번에 한 명만 (단일 writer 제약은 WAL에서도 유지)
- reader와 writer가 서로 막지 않음 (writer는 별도 WAL 파일에 append)

동시 쓰기가 거의 없는 환경이라도 "DB is locked" 류 에러를 피하려면 WAL을 켜두는 게 무난한 기본값이다. 단, WAL은 `.db-wal` / `.db-shm` 보조 파일이 생겨 단일 파일 백업 가정이 깨질 수 있다.

### 언제 SQLite가 맞는가

| 요구 | SQLite가 충족하는 방식 |
|------|----------------------|
| 운영 부담 0 | 데몬 없음. 설치 = 라이브러리 링크. 백업 = `.db` 파일 복사 |
| Repository 어댑터로 격리 | sqlite3 API를 어댑터 내부에 가두고, 도메인은 Repository 인터페이스만 의존 |
| 빠른 프로토타이핑 | `.db` 없으면 첫 호출 시 자동 생성. 스키마 마이그레이션도 단순 |
| 디버깅 | `sqlite3 data.db` CLI로 즉시 SQL 실행/스키마 확인 |
| 의존성 부담 최소화 | OS 시스템 라이브러리로 충분, vcpkg/Conan 회피 가능 |
| 신뢰성 | Public Domain + 100% 분기 커버리지 |

반대로 동시 쓰기가 많은 멀티 클라이언트 워크로드라면 client-server DB가 유리하다.

## 사용 예시

C/C++ 최소 사이클:

```cpp
#include <sqlite3.h>

sqlite3* db = nullptr;
int rc = sqlite3_open("data.db", &db);   // 없으면 생성, 있으면 열기
if (rc != SQLITE_OK) { /* 에러 처리 */ }

sqlite3_stmt* stmt = nullptr;
const char* sql = "INSERT INTO todo(title, due) VALUES(?, ?);";
sqlite3_prepare_v2(db, sql, -1, &stmt, nullptr);   // v1 prepare는 deprecated

sqlite3_bind_text(stmt, 1, "review skill", -1, SQLITE_TRANSIENT);
sqlite3_bind_text(stmt, 2, "2026-05-30",   -1, SQLITE_TRANSIENT);

rc = sqlite3_step(stmt);                 // 실행
if (rc != SQLITE_DONE) { /* 에러 처리 */ }

sqlite3_finalize(stmt);                  // 누락 시 메모리 누수
sqlite3_close(db);
```

핵심 규약: `sqlite3_prepare_v2` 사용, `sqlite3_finalize` 필수, 대량 INSERT는 트랜잭션으로 묶기(`BEGIN; ... COMMIT;` — 안 그러면 매 INSERT마다 fsync가 일어나 100~1000배 느려짐), 모든 반환 코드 확인(SQLITE_OK / SQLITE_DONE / SQLITE_ROW).

빌드(시스템 라이브러리): `sudo apt install libsqlite3-dev` 후 `g++ main.cpp -lsqlite3 -o app`.

## 흔한 오해

- "SQLite는 장난감 DB" — 아니다. Boeing 787 항공전자, iOS 시스템 DB, Firefox 북마크 등 무거운 환경에서 쓰인다. ACID, B-tree 인덱스, MVCC(WAL) 등 본격 RDBMS 기능을 갖춘다.
- "서버 DB보다 항상 느리다" — 단일 프로세스 내 R/W는 네트워크 왕복이 없어 오히려 SQLite가 빠르다. 단 동시 쓰기가 많으면 client-server가 유리.
- "sqlite3 바이너리를 별도로 띄워야 한다" — 아니다. 앱이 직접 `libsqlite3`를 호출하면 끝. CLI는 디버깅용일 뿐.
- "여러 프로세스가 같은 .db를 동시에 쓸 수 없다" — OS 파일 락 기반으로 가능은 하다. 다만 writer는 항상 한 명(WAL에서도 동일).
- "Public Domain이라 안전성이 의심된다" — 라이선스와 품질은 직교한다. SQLite는 100% 분기 커버리지로 상용 DB보다 엄격하게 검증된다.
- "Public Domain이라 아무나 PR할 수 있다" — 사용은 자유지만 contribution은 매우 엄격. 코어팀이 모든 코드를 직접 작성/검토하며 외부 기여를 거의 받지 않는다(그래서 Public Domain 유지가 가능).

## 채택 시 고려사항 (미결)

SQLite를 도입할 때 프로젝트와 무관하게 흔히 마주치는 미결 선택지.

- C++에서 sqlite3 C API를 직접 쓸지, 얇은 RAII 래퍼(SQLiteCpp, sqlite_orm)를 도입할지 — 추가 의존성 vs 수동 리소스 관리의 트레이드오프
- WAL을 기본으로 켤지 — 동시 읽기 이점 vs `.db-wal`/`.db-shm` 보조 파일로 단일 파일 백업 가정이 깨지는 트레이드오프
- 스키마 마이그레이션을 어떻게 다룰지 — 수동 `ALTER TABLE` vs 마이그레이션 스크립트 디렉토리 vs 외부 도구

## 참고 자료

- About / Architecture: <https://sqlite.org/about.html>
- C/C++ Interface 개요: <https://sqlite.org/cintro.html>
- C/C++ API Reference: <https://sqlite.org/capi3ref.html>
- Amalgamation: <https://sqlite.org/amalgamation.html>
- How To Compile SQLite: <https://sqlite.org/howtocompile.html>
- WAL: <https://sqlite.org/wal.html>
- File Locking And Concurrency: <https://sqlite.org/lockingv3.html>
- License Information: <https://sqlite.org/src/doc/trunk/LICENSE.md>
- How SQLite Is Tested: <https://sqlite.org/testing.html>
- TH3 Test Harness: <https://sqlite.org/th3.html>
- Official Git Mirror: <https://github.com/sqlite/sqlite>
- Fossil VCS: <https://fossil-scm.org/>
