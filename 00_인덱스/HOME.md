---
type: 인덱스
created: 2026-08-20
updated: 2026-08-21
---

# HOME

개인 위키의 시작점입니다.

## 빠른 시작

- 생각이나 할 일을 빠르게 기록: [[INBOX]]
- AI와 함께 작업할 때의 공용 규칙: [[AI_START]]
- 현재 진행 중인 일: [[프로젝트_인덱스]]
- 지속적으로 관리하는 영역: [[영역_인덱스]]
- 다시 사용할 지식: [[지식_인덱스]]
- 읽거나 본 자료: [[자료_인덱스]]

## 최근 수정 문서

아래 표는 VS Code의 Markdown 미리보기에서 Foam이 자동으로 생성합니다.

```foam-query
filter:
  not:
    path: "^/90_보관/"
select:
  - title
  - type
  - field: properties.updated
    label: 수정일
sort: properties.updated DESC
format: table
limit: 10
```

## 정리 순서

1. `INBOX`에 먼저 기록합니다.
2. 프로젝트·영역·지식·자료 중 알맞은 위치로 옮깁니다.
3. 관련 문서를 위키링크로 연결합니다.
4. Foam 저장 쿼리에서 태그와 문서 연결 상태를 점검합니다.
5. 사용이 끝난 문서는 `90_보관/`으로 옮깁니다.

> 토큰, 비밀번호, 개인정보와 공개하기 어려운 기록은 커밋하지 않습니다.
