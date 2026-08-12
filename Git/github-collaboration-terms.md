# GitHub 협업 용어: PR, Draft PR, MR, Fork

## 문제

GitHub로 브랜치를 병합하는 과정에서 다음 용어와 작업 흐름이 헷갈렸다.

- Pull Request와 Merge Request는 무엇이 다른가?
- Draft PR은 일반 PR과 무엇이 다른가?
- Fork, Clone, Branch는 각각 언제 사용하는가?
- 작업 브랜치의 변경을 `develop`에 반영하려면 어떤 흐름으로 진행하는가?

## 핵심 개념

### Pull Request와 Merge Request

두 용어 모두 **작업 브랜치의 변경을 대상 브랜치에 병합해 달라는 요청**을 의미한다. 서비스마다 이름만 다르다.

| 서비스 | 용어 |
| --- | --- |
| GitHub | Pull Request(PR) |
| GitLab | Merge Request(MR) |
| Bitbucket | Pull Request(PR) |

예를 들어 작업 브랜치의 변경을 `develop`에 반영하고 싶다면 GitHub에서는 `develop`을 대상으로 PR을 만든다고 표현한다.

### Draft PR

Draft PR은 아직 최종 검토나 병합 준비가 끝나지 않은 초안 상태의 PR이다. 변경 내용을 미리 공유하고 의견을 받을 수 있지만, `Ready for review`로 전환하기 전에는 병합할 수 없다.

### Fork, Clone, Branch

| 개념 | 의미 | 주 사용 상황 |
| --- | --- | --- |
| Fork | 다른 사람의 원격 저장소를 내 GitHub 계정으로 복사 | 직접 Push 권한이 없는 저장소에 기여할 때 |
| Clone | 원격 저장소를 내 컴퓨터로 내려받기 | 로컬에서 개발을 시작할 때 |
| Branch | 하나의 저장소 안에서 독립된 작업 흐름을 만들기 | 기능 개발, 버그 수정, 문서 작업을 분리할 때 |

## 원인

GitHub와 GitLab이 같은 협업 개념에 서로 다른 이름을 사용하고, Fork와 Clone 모두 저장소를 복사하는 동작처럼 보이기 때문에 혼동하기 쉽다. 또한 PR을 생성할 때 일반 PR과 Draft PR을 선택할 수 있어 각각의 목적을 모르면 차이가 불분명하다.

## 해결 방법

GitHub에서 `develop`을 중심으로 작업할 때는 다음 흐름을 사용한다.

```text
develop
  └─ feature/작업명 브랜치 생성
       └─ 코드 수정
            └─ commit
                 └─ 작업 브랜치를 원격에 push
                      └─ 작업 브랜치 → develop PR 생성
                           └─ 검토 후 merge
```

작업이 진행 중이라면 Draft PR로 먼저 공유하고, 완료되면 `Ready for review`로 전환한다. 본인이 소유하고 Push 권한이 있는 저장소에서는 보통 Fork 없이 Branch만 만들어 작업하면 된다.

반대로 오픈소스처럼 직접 Push할 수 없는 저장소에 기여할 때는 다음 흐름을 사용한다.

```text
원본 저장소 Fork → 내 계정의 저장소 Clone → Branch 생성
→ 수정·Commit·Push → 내 Fork에서 원본 저장소로 PR 생성
```

## 주의할 점

- GitHub에서는 Merge Request가 아니라 Pull Request라는 용어를 사용한다.
- Draft PR은 변경 사항을 공유하기 위한 초안이며, 준비 완료 상태로 바꾸기 전에는 병합할 수 없다.
- Fork는 GitHub 서버의 내 계정에 저장소를 복사하고, Clone은 저장소를 로컬 컴퓨터로 내려받는다.
- 새 작업 브랜치는 최신 대상 브랜치에서 생성해야 불필요한 충돌을 줄일 수 있다.
- PR을 만들 때 base 브랜치와 compare 브랜치의 방향을 반드시 확인한다.

## 한 줄 정리

> GitHub에서는 작업 브랜치를 PR로 대상 브랜치에 병합하며, 권한이 없는 외부 저장소에 기여할 때는 Fork를 사용한다.

## 참고 자료

- [GitHub Docs - About pull requests](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/proposing-changes-to-your-work-with-pull-requests/about-pull-requests)
- [GitHub Docs - About forks](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/about-permissions-and-visibility-of-forks)
