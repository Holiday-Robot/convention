# Go *Draft*

## 커뮤니티 표준

이 가이드는 아래 문서들을 기반으로 합니다. 아래에서 명시적으로 언급하지 않은 사항은 이 문서들을 따릅니다.

- [Effective Go](https://go.dev/doc/effective_go)
- [Go Code Review Comments](https://github.com/golang/go/wiki/CodeReviewComments)
- [Google Go Style Guide](https://google.github.io/styleguide/go/)


## 도구

### Formatter

| 도구                                                               | 설명                                                                  |
| ------------------------------------------------------------------ | --------------------------------------------------------------------- |
| [`gofmt`](https://pkg.go.dev/cmd/gofmt)                            | Go 표준 코드 포매터. Go 툴체인에 내장되어 있습니다.                   |
| [`goimports`](https://pkg.go.dev/golang.org/x/tools/cmd/goimports) | `gofmt`의 상위 집합으로, import 구문을 자동으로 추가·제거·정렬합니다. |

`goimports`를 기본 포매터로 사용합니다. `gofmt`의 모든 기능을 포함하므로 별도로 실행할 필요가 없습니다.

### Linter

[`golangci-lint`](https://golangci-lint.run/)를 사용합니다. 아래 linter를 활성화합니다.

| Linter                                                             | 설명                                         |
| ------------------------------------------------------------------ | -------------------------------------------- |
| [`errcheck`](https://github.com/kisielk/errcheck)                  | 무시된 error 반환값을 검출합니다.            |
| [`govet`](https://pkg.go.dev/cmd/vet)                              | `go vet`과 동일한 정적 검사를 수행합니다.    |
| [`staticcheck`](https://staticcheck.io/)                           | 버그, 성능 문제, 불필요한 코드를 검출합니다. |
| [`revive`](https://github.com/mgechev/revive)                      | Go 관용 코드 스타일을 검사합니다.            |
| [`goimports`](https://pkg.go.dev/golang.org/x/tools/cmd/goimports) | import 정렬 및 포매팅을 검사합니다.          |


## 스타일 규칙

포매터와 린터가 강제할 수 있는 규칙만을 정의합니다.

### Formatting

`gofmt`/`goimports`가 결정하는 사항(들여쓰기, 중괄호 위치, 공백 등)은 별도로 규정하지 않습니다.
포매터의 출력을 그대로 따릅니다.

### Import 그룹

`goimports`는 import를 아래 순서로 자동 그룹화합니다. 그룹 간에는 빈 줄로 구분됩니다.

```go
import (
    // 1. 표준 라이브러리
    "fmt"
    "os"

    // 2. 서드파티
    "github.com/pkg/errors"

    // 3. Holiday Robotics 패키지
    "hday.io/foo/internal/bar"
)
```

내부 패키지를 별도 그룹으로 분리하려면 `goimports`의 `-local` 플래그에 모듈 경로를 지정합니다.

```sh
goimports -local hday.io
```


## 설계 지침

도구로 강제할 수 없지만, 코드의 일관성과 가독성을 위해 따라야 할 지침입니다.

### 오류 처리

#### 오류에 문맥 추가

오류를 반환할 때는 `fmt.Errorf`와 `%w`로 문맥을 추가합니다.
호출 스택을 따라 올라가면서 각 레이어가 자신이 하려 했던 작업을 설명해야 합니다.

```go
// ❌ Bad
return err

// 🟢 Good
return fmt.Errorf("load user %d: %w", id, err)
```

#### 오류 메세지

오류 메시지는 소문자로 시작하고, "failed to"로 시작하지 않으며, 마침표로 끝나지 않습니다.

#### 센티널 오류와 오류 타입

패키지 외부에서 오류를 구분해야 한다면 센티널 오류(`var ErrXxx = errors.New(...)`) 또는 오류 타입(`type XxxError struct`)을 사용합니다.
단순한 메시지 전달이 목적이라면 `fmt.Errorf`로 충분합니다.

#### 오류 래핑

`errors.Is` / `errors.As`로 검사할 수 있도록 `%w`로 래핑합니다.
오류의 구체적인 내용을 숨겨야 하는 경우(예: 외부 API 응답)에만 `%v`를 사용합니다.

### 명명

#### 패키지 경로

패키지 경로는 `hday.io`로 시작합니다.

#### 식별자

공개 식별자는 표준 Go 명명 규칙을 따릅니다.
비공개 식별자는 지역 변수만 snake_case로 작성하고, 나머지는 camelCase로 작성합니다.

#### 패키지 이름

패키지 이름은 소문자 단일 단어를 사용합니다. 언더스코어나 복합어는 피합니다.
패키지 이름은 패키지가 제공하는 기능의 핵심을 나타내야 합니다.

```go
// ❌ Bad
package user_manager
package userManager

// 🟢 Good
package user
```

#### 인터페이스 이름

단일 메서드 인터페이스는 메서드 이름에 `-er` 접미사를 붙입니다.

```go
type Reader interface {
    Read(p []byte) (n int, err error)
}
```

다수의 메서드를 가진 인터페이스는 역할을 명확히 표현하는 명사를 사용합니다.

#### 변수 이름

짧은 스코프에서는 짧은 이름을, 넓은 스코프에서는 설명적인 이름을 사용합니다.
루프 인덱스 `i`, 에러 `err`, 컨텍스트 `ctx` 등 Go 커뮤니티의 관용적인 약어를 따릅니다.
