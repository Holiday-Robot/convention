# Protobuf *Draft*

## 커뮤니티 표준

이 가이드는 아래 문서들을 기반으로 합니다. 아래에서 명시적으로 언급하지 않은 사항은 이 문서들을 따릅니다.

- [Protocol Buffers Style Guide](https://protobuf.dev/programming-guides/style/)
- [Google API Design Guide](https://cloud.google.com/apis/design)
- [Buf Style Guide](https://buf.build/docs/best-practices/style-guide/)


## 도구

### Formatter

| 도구                               | 설명                                                  |
| ---------------------------------- | ----------------------------------------------------- |
| [`buf format`](https://buf.build/) | Protobuf 전용 포매터. buf 툴체인에 내장되어 있습니다. |

`buf format`을 기본 포매터로 사용합니다.

### Linter

| 도구                             | 설명                                                      |
| -------------------------------- | --------------------------------------------------------- |
| [`buf lint`](https://buf.build/) | Protobuf 정적 분석기. 명명 규칙과 설계 원칙을 검사합니다. |

`buf lint`를 사용합니다. 기본 rule set은 `DEFAULT`를 사용합니다.

```yaml
# buf.yaml
version: v2
lint:
  use:
    - DEFAULT
```


## 스타일 규칙

포매터와 린터가 강제할 수 있는 규칙만을 정의합니다.

### Formatting

`buf format`이 결정하는 사항(들여쓰기, 중괄호 위치, 공백 등)은 별도로 규정하지 않습니다.
포매터의 출력을 그대로 따릅니다.

### 명명

#### 파일

파일 이름은 `lower_snake_case.proto`를 사용합니다.

```
# ❌ Bad
UserService.proto
userservice.proto

# 🟢 Good
user_service.proto
```

#### 패키지

패키지 이름은 snake_case를 사용하며, 버전을 포함합니다.

```proto
// ❌ Bad
package hday.BehaviorTree;

// 🟢 Good
package hday.behavior_tree.v1;
```

#### 메시지와 열거형

메시지와 열거형 이름은 `PascalCase`를 사용합니다.

```proto
// ❌ Bad
message user_profile { ... }
enum user_status { ... }

// 🟢 Good
message UserProfile { ... }
enum UserStatus { ... }
```

#### 필드

필드 이름은 `snake_case`를 사용합니다.

```proto
// ❌ Bad
message User {
  string firstName = 1;
  string LastName = 2;
}

// 🟢 Good
message User {
  string first_name = 1;
  string last_name = 2;
}
```

#### 열거형 값

열거형 값은 `SCREAMING_SNAKE_CASE`를 사용하며, 열거형 이름을 접두사로 붙입니다.

```proto
// ❌ Bad
enum Status {
  UNKNOWN = 0;
  ACTIVE = 1;
  INACTIVE = 2;
}

// 🟢 Good
enum UserStatus {
  USER_STATUS_UNSPECIFIED = 0;
  USER_STATUS_ACTIVE = 1;
  USER_STATUS_INACTIVE = 2;
}
```

첫 번째 값은 반드시 `0`이며, `_UNSPECIFIED`로 끝나야 합니다.

#### 서비스와 RPC

서비스와 RPC 이름은 `PascalCase`를 사용합니다.

```proto
// ❌ Bad
service user_service {
  rpc get(UserGetRequest) returns (UserGetResponse);
}

// 🟢 Good
service UserService {
  rpc Get(UserGetRequest) returns (UserGetResponse);
}
```

RPC의 요청·응답 메시지는 `{Resource}{Action}Request`, `{Resource}{Action}Response` 형식을 따릅니다.
리소스 이름이 가장 앞에 오는 이유는 RPC가 리소스에 대한 행동을 표현하기 때문이며, 또한 개발 경험상 인텔리센스에서 리소스 이름이 먼저 나오는 것이 더 편리하기 때문입니다. 예를 들어 User를 생성하는 경우 UserService를 먼저 타이핑하고, 그 다음 행동을 타이핑하게 됩니다:

```go
User().Create(ctx, UserCreateRequest{...})
// vs
User().Create(ctx, CreateUserRequest{...}) 
```

메서드에는 리소스 이름을 포함하지 않습니다.
예를 들어 `UserCreate`가 아니라 `Create`입니다.
이에 대한 자세한 내용은 [서비스 설계](#서비스-설계) 섹션을 참고하세요.


## 설계 지침

도구로 강제할 수 없지만, API의 일관성과 안정성을 위해 따라야 할 지침입니다.

### 파일 구조

`.proto` 파일은 아래 순서로 구성합니다.

```proto
edition = "2023";

package hday.v1;

import "google/protobuf/timestamp.proto";

option go_package = "hday.io/gen/hday/user/v1;userv1";

// 서비스 정의
// 메시지 정의
```

### Edition

`syntax` 대신 `edition`을 사용합니다.
Protobuf Editions는 proto2/proto3를 장기적으로 대체하기 위해 도입된 새로운 방식으로, 파일 단위 또는 필드 단위로 동작 방식을 세밀하게 제어할 수 있습니다.

```proto
// ❌ Bad
syntax = "proto3";

// 🟢 Good
edition = "2023";
```

proto3의 기본 동작이 필요한 경우 별도의 feature 설정 없이 `edition = "2023"`만으로 충분합니다.

### 필드 번호

한 번 배포된 필드 번호는 재사용하지 않습니다.
필드를 삭제할 경우 `reserved`로 번호를 예약하여 실수로 재사용되는 것을 방지합니다.

```proto
message User {
  reserved 3, 5;
  reserved "email", "phone";

  string id = 1;
  string name = 2;
}
```

### 선택적 필드

`edition = "2023"`에서 단수 스칼라 필드의 기본 presence는 `EXPLICIT`입니다.
모든 필드가 설정 여부를 추적하므로, proto3에서 `optional`을 명시해야 했던 것과 달리 별도 선언 없이도 기본값과 미설정을 구별할 수 있습니다.

```proto
// edition 2023: 별도 선언 없이도 미설정과 기본값("")을 구별할 수 있음
message User {
  string display_name = 1;
}
```

반대로 proto3의 implicit presence(기본값과 미설정을 구별하지 않는) 동작이 필요하다면 `features.field_presence`를 명시합니다.

```proto
message Metric {
  string value = 1 [features.field_presence = IMPLICIT];
}
```

### 서비스 설계

서비스는 단일 리소스의 컬렉션으로 생각하고 컬렉션 또는 단일 리소스에 대한 행동을 표현하는 RPC를 정의합니다.

```proto
service UserService {
  rpc Create(UserCreateRequest) returns (User);
  rpc List(UserListRequest) returns (UserListResponse);
  rpc Get(UserGetRequest) returns (User);
  rpc Update(UserUpdateRequest) returns (UserUpdateResponse);
  rpc Delete(UserDeleteRequest) returns (google.protobuf.Empty);

  rpc Transfer(UserTransferRequest) returns (UserTransferResponse);
}
```

위 예시와 같이 어떤 리소스에 대한 서비스는 해당 리소스를 중심으로 응답되어야 합니다.
예를 들어 Library Service를 생각해봅시다.
`LibraryService.Get`은 `Book`을 응답할까요 아니면 `Author`를 응답할까요?
우리는 리소스를 중심으로 서비스를 설계하기때문에 서비스 이름이 `LibraryService`이므로 `Get`은 `Library`를 응답해야 합니다.
`Book`이나 `Author`는 별도의 서비스로 분리되어야 합니다.
이렇게 하면 API의 일관성과 예측 가능성이 높아집니다.

예시에서는 `Create`, `Get`이 리소스를 응답하지만, 필요에따라 추가 정보의 응답이 필요한 경우 `{Resource}{Action}Response` 메시지를 정의하여 응답할 수 있습니다.


#### Mutation

`XxxUpdate`는 리소스의 상태를 업데이트하는 RPC를 나타냅니다.
이는 리소스의 특수한 상태 변화를 표현하지 못하므로 Public API에서 사용되어서는 안됩니다.
예를 들어 `UserTransfer`는 `UserUpdate`보다 명확한 의도를 전달하므로 Public API에서 선호됩니다.
단, 테스트 등 내부 사용에서는 유연한 리소스 조작을 위해 `XxxUpdate` 정의가 필요합니다.
