---
title: "MCP 를 통한 workflow 자동화"
seoTitle: "MCP로 워크플로우 자동화 구현"
seoDescription: "Explore workflow automation with MCP, transforming AI-native companies and leveraging tools like Claude code for efficient task management"
datePublished: Sat Feb 14 2026 12:35:44 GMT+0000 (Coordinated Universal Time)
cuid: cmlmav58e000802ieg32gdky3
slug: mcp-workflow
ogImage: https://cdn.hashnode.com/res/hashnode/image/upload/v1771072529291/38d1d9ca-2333-4fe9-82b5-824dbee31697.png

---

## AI native

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771071648205/95232f7d-c89e-42f8-9d96-dfdbdc40cb44.png align="center")

최근에 LinkedIn 이나 여러 소셜 플랫폼들의 글을 보면 **AI native 회사** 라는 워딩들이 많이 보입니다. IBM 의 정의에 따르면 AI native 를 아래와 같이 정의한다고 하는데요.

> *“****AI를 사고와 업무 방식에 끊임없이 내재화하는 상태****”*

그렇다면 팀원들이 계속해서 AI 를 사고와 업무 방식에 끊임 없이 내재화 하려면 어떻게 해야할까요? 개발자들은 이미 Claude code 나 Codex 등 여러 AI Tool 들을 사용하는데 익숙하지만 대부분의 비개발자 직군의 사람들은 별도로 공부를 하지 않는다면 사용하는 사람들을 찾기 어렵습니다. 또한 금전적인 문제 또한 이 분야에 가장 큰 이슈이기도 하기 때문입니다.

이러한 생각을 하다 사내에서 비개발자 직군분들이 Claude code 를 알려주면 잘 쓸수 있을까? 라는 고민을 하기 시작했고, 이를 기반으로 Claude code 를 어떻게 사용해야 하는지에 대한 **아래처럼 장표를 만들고 이를 공유하는 자리**를 가지게 되었습니다.

## Workflow 로서의 Claude code

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771061887814/782a4e4d-747b-4084-b9b4-44d635d73833.png align="center")

세션 진행간 Terminal 보다는 GUI 기반의 Google Antigravity 를 통해 Claude code 를 이용하도록 했고, Context 와 SKILL 의 개념 그리고 워크플로우에 어떻게 적용해야 하는지 등등을 공유했습니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771067700669/b37920b8-b5a5-41dd-9334-2e769e2d2c55.png align="center")

세션에서 가장 중요하게 공유했던 부분은 **SKILL** 에 관한 부분인데요. 이 SKILL 을 가장 중요하게 생각하는 이유는 **개인이 특정한 Task 를 수행할 때 수행하는 일련의 작업이나 지식들을 담는 곳** 이기 때문입니다. claude code 와 같은 도구들은 이미 많이 SOTA 모델을 잘 오케스트레이션하여 사용하여 이미 똑똑하지만 학습되지 않은 연속적인 작업들을 수행하는데는 어려움을 겪으므로 SKILL 을 통해 지식을 전수해줘야 하기 때문입니다.

그래서 SKILL 의 예시로 세션에서 `playwright` 를 통해 브라우저 자동화를 하는 방법에 대해 공유하였는데 이후에 어드민의 일부 작업들을 `playwright` 를 통해 자동화 하기 시작했다고 말씀해주셨습니다. 자동화 해준 이야기들을 듣다보니 계속해서 아래와 같은 고민이 들기 시작했습니다.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1771068660722/c94ba813-7bd3-4fb0-ac19-e7d7f56c24a6.png align="center")

> 우리가 기존에 운영하던 내부 어드민은 정말 HTML 로 계속해서 유지되어야 하나?

사실 특정 작업들을 AI 도구들이 쉽게 작업하도록 도와주기 위해서는 ***“기존의 사람 친화적인 interface 보다는 LLM Model 에 친화적인 Interface 가 낫지않을까?”*** 라는 생각이 들었습니다. 기존 HTML 형식의 GUI 는 사람에게 정보를 정리해서 가독성이 좋게 전달하기 위한 수단인데, 사실 LLM 에게는 그것보단 구조화된 JSON, YAML 등의 형태가 더 이해하기 쉬울수도 있겠다 라는 생각이 들었습니다.

그래서 **AI 가 필요한 정보나 액션만 취할 수 있는 도구를 쥐어준다는 느낌으로** `mcp` **로 제공해주면 어떨까?** 라는 생각이 들었습니다. 그래서 `mcp` 로 어드민에서 사람에게 보여주던 정보들을 기존의 API 를 통해 이용할 수 있도록 내부에 배포하여 이용할수 있도록 하였더니 꽤나 이것들을 이용해서 이것저것 많이 시도해보시는 느낌을 받았습니다.

이러한 현상을 보면서 AI native 로 변해간다는건 조직원들이 AI Tool 에 대해 익숙해지고 더 많이 사용하게 되고, 원래 사람위주의 Interface 들이 점차 **AI native 하게 정말 구조적인 정보 또는 액션만 제공하는 구조로 워크플로우 자체가 변해가는 것이구나** 라는 생각이 들었습니다. 위 작업을 하면서 기존에는 크게 잘 사용하지 않던 `mcp` 에 대한 시각도 많이 바뀌게 된거 같습니다.

## 마치며

아직은 AI 초기라 잘 모르겠지만 이후에는 모두가 Claude code 와 같은 에이전트 형태의 커스터마이징이 가능한 Tool 을 이용하게 될 것이고 **이제 항상 반복되던 Admin 작업들은 LLM 성능이 좋아지면 좋아질수록, 잘 관리된 도메인 지식인 SKILL 과 내부 데이터 및 액션에 대한 도구를 제공해주는** `mcp` **로 자동화 하는 방식으로 발전해나갈 것**이라는 생각이 들었습니다.

개인적으로 비개발자 분들이 여러 업무에 자동화를 하는 것을 보면서 제 개인적으로도 인싸이트를 많이 얻었던 것 같습니다. Mixpanel 의 로그를 분석한다거나, 동영상 편집에 이용한다거나 등등. 이 글을 읽으시는 개발자 분들도 일상생활을 자동화 하기위해 클로드 코드를 많이 이용해보시길 바랍니다.