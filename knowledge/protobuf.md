---
kind: concept
id: protobuf
title: Protocol Buffers
summary: Protocol Buffers는 구조화된 데이터를 언어·플랫폼 중립적으로 직렬화하는 Google의 IDL + 바이너리 인코딩 + 코드 생성 체계다.
category: serialization
tags: [protobuf, serialization, binary-encoding, idl, wire-format, schema, google, proto3]
relations: [contrasts_with:json, same_family:avro, same_family:thrift, same_family:flatbuffers, related:grpc, related:schema-evolution]
---

## 개요

세 층으로 이해하면 깔끔하다.

- **스키마 언어(IDL)**: `.proto` 파일에 메시지(레코드) 구조를 선언한다. 사람이 쓰는 계약서에 해당.
- **코드 생성**: `protoc` 컴파일러가 `.proto`를 읽어 대상 언어의 클래스/구조체와 직렬화 코드를 생성한다.
- **와이어 포맷**: 실제로 바이트로 나갈 때의 바이너리 인코딩 규약. 작고 파싱이 빠른 것이 목적.

JSON·XML처럼 자기 기술적(self-describing) 텍스트가 아니라, 스키마를 양쪽이 미리 공유한다는 전제의 바이너리 포맷이다.<br>
그래서 더 작고 빠르지만, 스키마 없이는 바이트만 봐서 의미를 복원하기 어렵다.

## `.proto` 스키마

```proto
syntax = "proto3";

message Person {
  string name            = 1;
  int32  id              = 2;
  repeated string emails = 3;
}
```

핵심은 각 필드 뒤의 **필드 번호(field number, tag)**다.

- 필드 번호는 와이어에서 필드를 식별하는 키다.
  - 필드 이름은 와이어에 실리지 않고 번호만 나간다.
- 한번 배포되면 번호를 바꾸면 안 된다. 이름을 바꿔도 와이어 호환성에 영향 없지만, 번호를 바꾸면 다른 필드로 오인된다.
- 1~15번은 태그가 1바이트로 인코딩되므로 자주 쓰는 필드에 배정하는 것이 유리하다(16 이상은 2바이트 이상).

## 와이어 포맷

각 필드는 (태그, 값) 쌍으로 직렬화된다.<br>
태그 바이트 안에 필드 번호와 wire type이 함께 들어간다.

```
tag = (field_number << 3) | wire_type
```

| wire type | 이름 | 용도 |
|-----------|------|------|
| 0 | VARINT | int32, int64, uint, bool, enum 등 가변 길이 정수 |
| 1 | I64 | fixed64, double |
| 2 | LEN | string, bytes, 내장 메시지, packed repeated |
| 3 | SGROUP | 그룹 시작 -- **deprecated, 사용 금지** |
| 4 | EGROUP | 그룹 종료 -- **deprecated, 사용 금지** |
| 5 | I32 | fixed32, float |

- **Varint**
  - 정수를 7비트씩 끊어 바이트 최상위 비트를 "다음 바이트 있음" 표시로 쓰는 가변 길이 인코딩이다.
  - 작은 수일수록 적은 바이트를 쓴다.

- **ZigZag 인코딩**: 부호 있는 정수를 그냥 varint로 넣으면 음수가 항상 최대 길이(보통 10바이트)로 부풀기 때문에,
  - `sint32`/`sint64`는 ZigZag로 부호를 짝/홀로 매핑해 작은 절댓값을 작은 바이트로 만든다.
  - 음수를 자주 쓴다면 `int32` 대신 `sint32`가 유리하다.

- **JSON과 크기 비교**: 같은 데이터 "id 필드에 값 150"을 각각 인코딩하면 차이가 드러난다.
  - JSON `{"id":150}`: 필드 이름 `id`를 매번 문자열로 전송한다. 따옴표·콜론·중괄호까지 합쳐 **10바이트**.
  - protobuf: 스키마에서 `id`가 2번 필드로 약속되어 있으므로, 와이어에는 이름 대신 번호만 실린다. "2번 필드, 값 150"을 바이너리로 표현하면 **3바이트**.

protobuf가 작은 이유는 결국 하나다 -- 필드 이름을 매번 보내지 않고, 사전에 약속한 번호로 대체하기 때문이다.

## 호환성과 스키마 진화

수신측 스키마에 없는 필드 번호를 만나면, 파서는 wire type 정보로 그 필드를 건너뛸 수 있다(길이를 알기 때문).<br>
이것이 전방/후방 호환의 토대다.

- **후방 호환** (새 스키마가 옛 데이터 읽기): 필드 추가는 안전하다. 옛 데이터에 없던 필드는 기본값으로 채워진다.
- **전방 호환** (옛 스키마가 새 데이터 읽기): 모르는 필드는 무시(또는 보존)되고 나머지는 정상 파싱된다.

진화 규칙 요지:

- 필드 번호 재사용 금지
- 타입 임의 변경 금지 (호환되는 일부 변경만 허용)
- 삭제한 번호는 `reserved`로 막아 재사용을 방지

## proto2 vs proto3

- proto3가 현재 기본. 문법이 단순화됐다.
- proto2의 `required`는 "한번 required면 영원히 빼기 어렵다"는 이유로 proto3에서 제거됐다.
- proto3는 스칼라 필드에 명시적 존재(presence) 개념이 기본적으로 약했다.
  - 기본값(0, 빈 문자열)과 "값 없음"을 구분하기 어려웠다.
  - 이후 `optional` 키워드가 복귀해 현재 공식 문서에서 권장 방식으로 제시된다
- syntax 기반(proto2/proto3) 대신 기능을 개별 토글하는 **Editions** 모델이 도입되고 있다.

## 흔한 오해

> **오해:** "protobuf는 자기 기술적이다"
> - 와이어에는 필드 번호와 wire type만 있고, 필드 이름·타입 의미는 스키마가 있어야 복원된다.
> - 스키마 없이 바이트만으로는 사람이 읽기 어렵다.

> **오해:** "필드 이름을 바꾸면 호환성이 깨진다"
> - 와이어 호환성은 번호 기준이라 이름 변경은 영향이 없다 (생성 코드의 접근자 이름만 바뀜).
>
> > **반례:** 번호를 그대로 두고 `name`을 `full_name`으로 바꾼 `.proto`를 배포해도 기존 클라이언트는 정상 동작한다.

> **오해:** "JSON보다 항상 빠르고 작다"
> - 일반적으로 그렇지만, 가시성·디버깅·스키마 공유·버전 관리라는 트레이드오프가 있다.
> - 텍스트 가시성이 중요한 곳엔 JSON이 나을 수 있다.

> **오해:** "메시지를 이어붙이면(concatenate) 깨진다"
> - 같은 타입의 직렬화 바이트를 이어붙이면, 마지막 값 우선/repeated 병합 규칙에 따라
>   병합된 메시지로 파싱된다 (의도된 성질).

> **오해:** "필드를 지우면 끝"
> - 번호를 비워두고 나중에 재사용하면 옛 데이터와 충돌한다.
> - `reserved`로 막아야 한다.

## 미해결 질문

- **Editions 기본화 시점**: 2023·2024 Edition 명세는 존재하지만,
  proto2/proto3를 완전히 대체하는 기본이 되는 시점은 공식 문서에서 확인되지 않았다.
- **proto3 `optional` 복귀 버전**: 현재 권장 방식으로 제시되나 도입 버전은 명시되지 않았다.

## 참고 자료

- <https://protobuf.dev/> -- Protocol Buffers 공식 문서 (Overview, Programming Guides, Encoding)
- <https://protobuf.dev/programming-guides/encoding/> -- 와이어 포맷 상세
- <https://protobuf.dev/programming-guides/proto3/> -- proto3 언어 가이드
