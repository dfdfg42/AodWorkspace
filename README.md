# AodWorkspace

AOD (All of Dopamine) 프로젝트 워크스페이스. 영화/OTT/게임/웹툰/웹소설 정보를 여러 플랫폼에서 크롤링·집계해 AI 맞춤 추천을 제공하는 서비스의 저장소 3개를 서브모듈로 묶고, 저장소 밖에 흩어져 있던 아키텍처 문서와 블로그 글을 모아둔다.

각 서브모듈은 org `AOD-All-of-Dopamine`의 독립 저장소 그대로다. PR, CI, 배포는 각 저장소에서 하던 대로 하면 되고, 이 워크스페이스는 "어느 시점의 조합"을 기록할 뿐이다.

## 구성

| 경로 | 내용 | 원본 |
|---|---|---|
| `back/` | 백엔드 (Java/Spring, 멀티모듈, 잡 큐 크롤러) | [-AOD-All-of-Dopamine-back](https://github.com/AOD-All-of-Dopamine/-AOD-All-of-Dopamine-back) |
| `front/` | 프론트엔드 (TypeScript) | [-AOD-All-of-Dopamine-front](https://github.com/AOD-All-of-Dopamine/-AOD-All-of-Dopamine-front) |
| `ai/` | AI 추천 | [-AOD-All-of-Dopamine-AI](https://github.com/AOD-All-of-Dopamine/-AOD-All-of-Dopamine-AI) |
| `docs/` | 시스템·백엔드 아키텍처 다이어그램 (svg, jpg) | 워크스페이스 자체 |
| `blog/` | 기술 글 (탐색 필터 20초 → 1초 최적화) | 워크스페이스 자체 |
| `CLAUDE.md` | Claude Code 작업 가이드 (세 저장소 공통 진입점) | 워크스페이스 자체 |

## 서브모듈 사용법

처음 받을 때 (서브모듈까지 한 번에):

```bash
git clone --recurse-submodules https://github.com/dfdfg42/AodWorkspace
```

이미 받았는데 서브모듈 폴더가 비어 있으면:

```bash
git submodule update --init --recursive
```

세 저장소를 각자 최신 `main`으로 올리고 그 조합을 워크스페이스에 기록:

```bash
git submodule update --remote --merge
git add back front ai
git commit -m "chore: 서브모듈 포인터 갱신"
```

서브모듈 안에서 작업할 때는 그 폴더가 그냥 해당 저장소의 클론이다. `cd back` 후 평소처럼 브랜치 만들고 커밋하고 push하면 org 저장소로 간다. 워크스페이스 쪽에는 "back이 가리키는 커밋이 바뀌었다"만 잡히므로, 기록하고 싶으면 위처럼 `git add back` 후 커밋한다.

주의: 서브모듈은 기본적으로 detached HEAD 상태로 체크아웃된다. 안에서 커밋하기 전에 `git checkout main` 또는 작업 브랜치로 옮긴 뒤 시작한다.
