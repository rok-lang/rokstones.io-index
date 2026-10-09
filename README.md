# rokstones.io-index

[rok](https://rokstones.io) 의 패키지 (**stone**) 레지스트리 인덱스입니다. crates.io-index 와 같은 방식으로, 이 git 저장소 자체가 레지스트리입니다. 소스 tarball 을 올리는 서버는 없습니다: 각 버전은 원래 git 저장소의 커밋을 가리키고, `rok` 는 그 커밋을 가져와 tree hash 로 검증합니다.

웹사이트 / API: <https://rokstones.io>

## 배치

```text
config.rokon          레지스트리 설정
1/<name>              이름이 1 글자인 stone
2/<name>              2 글자
3/<c>/<name>          3 글자 (<c>: 첫 글자)
<ab>/<cd>/<name>      4 글자 이상 (<ab>: 1-2 번째 글자, <cd>: 3-4 번째 글자)
```

경로는 이름을 소문자로 바꾼 것입니다 (`Geom` → `ge/om/geom`). 대소문자만 다른 두 stone 은 같은 파일이 되므로 둘째는 publish 할 수 없습니다.

## `config.rokon`

```rokon
Registry {
    name: "rokstones.io",        // 레지스트리 이름
    dl: "git",                   // 소스 정책: 각 버전의 `source` (git 커밋) 에서 가져옴
    api: "https://rokstones.io", // 웹 / API
}
```

## stone 파일

파일 하나가 stone 하나이고, 줄 하나가 버전 하나입니다 (publish 순서, 오래된 것부터). 각 줄은 한 줄짜리 정규 형식 RokON 값 `Stone { .. }` 입니다. 빈 줄은 무시합니다.

```rokon
Stone { name: "geom", vers: "0.3.1", deps: [{ name: "num", req: "^1.2", kind: "normal" }], source: Dep::git("https://github.com/someone/geom", rev: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123"), cksum: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3", yanked: false }
```

| 필드     | 뜻                                                                                       |
|----------|------------------------------------------------------------------------------------------|
| `name`   | stone 이름 (식별자: 영문자, 숫자, `_`; 숫자로 시작하지 않음). `use <name>::..` 의 이름    |
| `vers`   | semver 버전 (`major.minor.patch[-pre][+build]`)                                          |
| `deps`   | 의존 stone 목록 (아래). 없으면 `[]`                                                      |
| `source` | `Dep::git("<url>", rev: "<커밋 40 자리>")`. 저장소 루트가 패키지 (`lib.rok`, `rok.rokon`) |
| `cksum`  | 그 커밋의 git tree hash. `rok` 가 checkout 후 비교하며, 다르면 에러                      |
| `yanked` | `true` 면 새 해소에서 고르지 않음 (이미 `rok.lock` 에 고정된 것은 그대로 씀)             |

`deps` 의 각 항목:

| 필드       | 뜻                                                                                      |
|------------|-----------------------------------------------------------------------------------------|
| `name`     | 의존 stone 이름                                                                         |
| `req`      | 버전 요구 (`^1.2`, `~1.2.3`, `=1.0.0`, `>=1.2, <2`, `1.*`, `*`; 연산자 없는 버전은 `^`) |
| `kind`     | `"normal"` 또는 `"dev"` (dev 는 그 stone 자신의 테스트용이며 따라가지 않음)             |
| `registry` | 선택. 다른 인덱스의 url. 없으면 이 인덱스                                               |

stone 의 의존은 stone 뿐입니다. `Dep::path` / `Dep::git` 의존이 있는 패키지는 publish 할 수 없고, path / git dev 의존은 항목에서 빠집니다.

## publish / yank

서버 없이, 이 저장소에 커밋을 더하는 것이 publish 입니다.

```
rok publish                                          # 항목 한 줄을 출력 (dry run)
rok publish --index <이 저장소 checkout> --commit    # 파일에 줄을 더하고 커밋
rok yank --vers 0.3.1 --index <checkout> [name]      # yanked: true 로 바꾸고 커밋 (--undo: 되돌림)
```

`rok publish` 는 현재 패키지의 `rok.rokon` (`name`, `version`, `deps`), git remote `origin` 의 url, `HEAD` 커밋과 tree hash 로 항목을 만듭니다. work tree 가 깨끗해야 하고, 패키지가 저장소 루트여야 하며, 같은 버전이 이미 있으면 에러입니다. 커밋을 `origin` 에 push 하는 것은 publish 하는 사람의 몫입니다. 만든 인덱스 커밋은 이 저장소에 pull request 로 보냅니다.

## 사용

`rok` 는 이 저장소를 `$ROK_HOME/cache/registry/db/` 에 bare clone 해 두고 읽습니다. 인덱스 위치는 `ROK_REGISTRY` 환경 변수, 없으면 `$ROK_HOME/config.rokon` 의 `Config { registry: "<url>" }`, 없으면 기본값 (이 저장소의 GitHub url) 입니다. 의존 하나만 다른 인덱스를 쓰려면 `Dep::version("^1.2", registry: "<url>")` 입니다.
