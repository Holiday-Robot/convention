# Shell *Draft*

## 커뮤니티 표준

이 가이드는 아래 문서를 기반으로 합니다. 아래에서 명시적으로 언급하지 않은 사항은 이 문서를 따릅니다.

- [Google Shell Style Guide](https://google.github.io/styleguide/shellguide.html)


## 도구

### Formatter

| 도구                                   | 설명                                                               |
| -------------------------------------- | ------------------------------------------------------------------ |
| [`shfmt`](https://github.com/mvdan/sh) | Shell 스크립트 포매터. bash, sh, mksh 등 다양한 방언을 지원합니다. |

`shfmt`를 기본 포매터로 사용하며, 아래 옵션을 적용합니다.

```sh
shfmt -i 0 -bn -ci -sr
```

| 옵션   | 설명                                             |
| ------ | ------------------------------------------------ |
| `-i 0` | 들여쓰기 탭                                      |
| `-bn`  | `&&` / `\|\|` 이항 연산자를 줄 끝이 아닌 줄 앞에 |
| `-ci`  | `case` 항목을 들여쓰기                           |
| `-sr`  | Redirection 연산자 앞에 공백 추가                |

### Checker

| 도구                                        | 설명                                                         |
| ------------------------------------------- | ------------------------------------------------------------ |
| [`shellcheck`](https://www.shellcheck.net/) | Shell 스크립트 정적 분석기. 버그와 위험한 패턴을 검출합니다. |

`shellcheck`를 사용합니다. 경고를 오류로 취급하며, CI에서 통과하지 않으면 merge할 수 없습니다.


## 스타일 규칙

포매터와 체커가 강제할 수 있는 규칙만을 정의합니다.

### Formatting

`shfmt`가 결정하는 사항(들여쓰기, 공백, 줄바꿈 등)은 별도로 규정하지 않습니다.
포매터의 출력을 그대로 따릅니다.

### Shebang

스크립트 첫 줄에 반드시 shebang을 명시합니다. `sh`가 아닌 `bash`를 사용합니다.

```sh
#!/usr/bin/env bash
```

`/bin/bash` 대신 `env`를 사용하는 이유는, 환경마다 bash의 경로가 다를 수 있기 때문입니다.


## 설계 지침

도구로 강제할 수 없지만, 코드의 일관성과 안전성을 위해 따라야 할 지침입니다.

### POSIX 호환성

가능한 한 POSIX 호환 문법을 사용합니다.
shebang은 `bash`로 고정하지만, bash 전용 기능은 명확한 이유가 있을 때에만 사용합니다.
이후 섹션에서 특정 문법을 권장하는 이유 중 상당수는 이 원칙에서 비롯됩니다.
만약 POSIX 호환 작성이 너무 복잡하거나 비효율적이라면, shell 스크립트가 아니라 다른 언어를 사용해야하는 것은 아닌지 고려해야 합니다.

### 언제 Shell을 사용하는가

Shell 스크립트는 명령어를 연결하고 파일 시스템을 조작하는 단순한 자동화 작업에 적합합니다.
아래 상황 중 하나라도 해당된다면 Python 등 다른 언어를 사용합니다.

- 100줄을 초과하는 경우
- 복잡한 데이터 구조가 필요한 경우
- 오류 처리가 복잡한 경우

### 안전 옵션

모든 스크립트는 shebang 다음 줄에 아래 옵션을 설정합니다.

```sh
#!/usr/bin/env bash
set -euo pipefail
```

| 옵션          | 설명                                                                   |
| ------------- | ---------------------------------------------------------------------- |
| `-e`          | 명령어가 실패하면 즉시 종료합니다.                                     |
| `-u`          | 정의되지 않은 변수를 참조하면 오류로 처리합니다.                       |
| `-o pipefail` | 파이프라인 중간 명령어가 실패하면 파이프라인 전체를 실패로 처리합니다. |

### 오류 처리

#### 오류 메시지

오류 메시지는 표준 오류(`stderr`)로 출력하고, 0이 아닌 종료 코드로 종료합니다.

```sh
# ❌ Bad
echo "ERROR: file not found"
exit 1

# 🟢 Good
echo "ERROR: file not found" >&2
exit 1
```

#### 정리 작업

임시 파일이나 리소스를 생성했다면 `trap`으로 정리합니다.

```sh
tmp=$(mktemp)
trap 'rm -f "$tmp"' EXIT
```

### 명명

#### 함수

함수 이름은 `snake_case`를 사용합니다. `function` 키워드는 생략합니다.

```sh
# ❌ Bad
function doSomething() { ... }

# 🟢 Good
do_something() { ... }
```

#### 변수

전역 상수는 `UPPER_SNAKE_CASE`, 지역 변수와 함수 인자는 `lower_snake_case`를 사용합니다.

```sh
readonly MAX_RETRY=3

process_file() {
    local input_file=$1
    ...
}
```

함수 내에서는 `local`로 변수 스코프를 제한합니다.

### 변수 참조

변수는 반드시 큰따옴표로 감쌉니다. 공백이나 특수문자로 인한 단어 분리(word splitting)를 방지합니다.

```sh
# ❌ Bad
cp $src $dst

# 🟢 Good
cp "$src" "$dst"
```

변수 이름은 `${}`로 감싸 경계를 명확히 합니다.

```sh
# ❌ Bad
echo "$filename_backup"   # filename_backup 변수를 참조

# 🟢 Good
echo "${filename}_backup"
```

### 옵션 이름

명령어 옵션은 가능한 한 축약형 대신 긴 이름(`--option`)을 사용합니다.
스크립트 내에서는 명령어가 어떤 동작을 하는지 명확히 드러나는 것이 중요합니다.

```sh
# ❌ Bad
kubectl -n foo get po -o wide

# 🟢 Good
kubectl --namespace foo get pods --output wide
```

...하지만 널리 알려진 옵션의 경우 축약형을 사용하는것이 더 가독성이 좋을 수 있습니다.

| 옵션      | 아마도        | 설명      |
| --------- | ------------- | --------- |
| `-f`      | `--force`     | 강제 실행 |
| `-r` `-R` | `--recursive` | 재귀 처리 |
| `-o`      | `--output`    | 출력 파일 |
| `-v`      | `--verbose`   | 상세 출력 |




### 명령어 치환

명령어 치환은 백틱(`` ` ``) 대신 `$()`를 사용합니다.

```sh
# ❌ Bad
output=`date +%Y%m%d`

# 🟢 Good
output=$(date +%Y%m%d)
```

### 조건문

문자열 비교와 파일 검사는 `[[ ]]`를 사용합니다. `[ ]`는 사용하지 않습니다.

```sh
# ❌ Bad
if [ "$name" = "foo" ]; then ...
if [ -f "$file" ]; then ...

# 🟢 Good
if [[ "$name" == "foo" ]]; then ...
if [[ -f "$file" ]]; then ...
```

### 스크립트 구조

실행 가능한 스크립트는 `main` 함수를 정의하고, 파일 끝에서 호출합니다.
최상위 코드를 최소화하여 `source`로 불러올 때 부작용이 없도록 합니다.

```sh
#!/usr/bin/env bash
set -euo pipefail

main() {
    ...
}

main "$@"
```
