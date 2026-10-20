---
icon: fas fa-box-open
order: 5
title: FileButler 빌드로그
---

로컬 LLM으로 내 PC의 파일을 분류하고 정리하는 에이전트를 만들고 있습니다. 만들면서 겪은 것을 순서대로 기록합니다.

**모든 판단은 로컬에서 합니다.** 파일 이름과 내용이 외부로 나가지 않습니다. 그리고 LLM은 "이 파일이 무엇인가"만 정하고, **실제로 파일을 옮기는 결정은 코드가** 합니다. 이 경계가 이 프로젝트의 중심이고, 글마다 그 경계를 어디까지 믿을 수 있는지 시험한 기록이 남습니다.

{% assign series = site.posts | where_exp: "post", "post.tags contains 'filebutler'" | sort: "date" %}

{% for post in series %}
### [{{ post.title }}]({{ post.url | relative_url }})

<small>{{ post.date | date: "%Y년 %m월 %d일" }}</small>

{{ post.summary }}

{% endfor %}

---

## 지금까지의 흐름

처음에는 LLM을 한 번 부르고 끝나는 단순한 스크립트였습니다. 거기서 **에이전트**(모델이 필요한 정보를 스스로 찾아보고 판단하는 방식)로 바꿨고, 조회 기능을 **MCP 서버**로 떼어내 다른 프로그램도 쓸 수 있게 했고, 지금은 **되돌리기**를 붙였습니다.

되돌리기가 생기면서 다음 단계가 열렸습니다. 지금까지 MCP 서버는 "읽기"만 할 수 있었는데, 잘못 옮겼을 때 수습할 방법이 생겼으니 **파일을 실제로 옮기는 권한**을 거기 내보낼 수 있습니다.

## 쓰는 기술

| | |
|---|---|
| 모델 | `qwen3.5:4b` (Ollama, 로컬) |
| 언어 | Python 3.12 |
| 구조화 출력 | Pydantic |
| 툴 연결 | MCP (Model Context Protocol) |
| 환경 | Windows |
