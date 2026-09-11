---
title: LangChain.js로 DB와 API를 사용하는 Agent 만들기
description: SQLite 조회와 날씨 API를 Tool로 연결하며 살펴보는 LLM Agent의 실행 흐름
categories: [LLM]
tags: [llm, agent, langchain, javascript, sqlite]
comments: true
toc: true
---

## 왜 이 예제를 만들었나

동료 개발자들에게 LLM의 Agent와 Tool을 설명할 일이 있었다. 개념만 이야기하기보다는, 익숙한 데이터베이스 조회와 API 호출을 연결해서 보여주면 이해하기 쉬울 것 같았다.

이번 글에서는 Node.js와 LangChain.js로 다음 질문에 답하는 작은 Agent를 만들어본다.

> 개발팀 구성원과 서울 날씨를 알려줘.

직원 정보는 SQLite에서 읽고, 날씨는 별도의 함수에서 가져온다. 글에 사용하는 직원과 조직 정보는 모두 가상 데이터다. 기존 교육용 예제를 두 개의 Tool로 줄이고, 실행에 필요한 파일을 한 폴더에 모았다.

## LLM, Tool, Agent가 각각 하는 일

이 예제에서 모델에 전달되는 것은 질문과 Tool의 이름·설명·입력 형식이다. LLM 자체가 SQLite 파일을 열거나 HTTP 요청을 실행하는 것은 아니다.

| 구성 요소 | 이 예제에서 맡는 역할 |
|---|---|
| LLM | 질문을 해석하고 사용할 Tool과 인자를 선택한다. 반환된 결과를 바탕으로 답변한다. |
| Tool | 개발자가 작성한 DB 조회 함수 또는 날씨 조회 함수를 실행한다. |
| Agent 실행부 | 모델의 Tool 호출 요청을 실행하고 결과를 다시 모델에 전달하는 흐름을 관리한다. |

LangChain의 `createAgent`는 모델과 Tool을 연결하는 실행 흐름을 제공한다. Tool에는 함수뿐 아니라 모델이 용도를 판단할 수 있는 설명과 입력 스키마를 함께 지정한다. [Agents 문서](https://docs.langchain.com/oss/javascript/langchain/agents), [Tools 문서](https://docs.langchain.com/oss/javascript/langchain/tools)

```text
사용자 질문
    ↓
LLM: 필요한 Tool과 인자 선택
    ↓
Agent 실행부: 선택한 함수 실행
    ├─ get_team_members → SQLite 조회
    └─ get_weather      → 테스트 응답 또는 날씨 API 호출
    ↓
Tool 결과를 LLM에 전달
    ↓
필요하면 추가 Tool 호출, 충분하면 최종 답변
```

두 Tool의 호출 순서나 한 번에 호출하는 개수는 모델의 판단에 따라 달라질 수 있다. 개발자가 `if`문으로 모든 질문을 분류하는 대신, 모델이 우리가 제공한 기능 중 필요한 것을 선택하게 하는 구성이다.

## 개발 환경과 파일 구성

예제 작성 시 로컬에 설치된 라이브러리 버전은 아래와 같다. Node.js는 26.8.2 환경에서 코드와 Tool을 확인했다.

| 라이브러리 | 버전 |
|---|---|
| langchain | 1.5.11 |
| @langchain/openai | 1.5.12 |
| better-sqlite3 | 12.11.1 |
| zod | 3.25.76 |

```text
agent-demo/
├── package.json
├── .env
├── .gitignore
├── init-db.js
├── tools.js
├── agent.js
└── company.db        # init-db.js 실행 시 생성
```

빈 폴더에서 다음 명령으로 시작한다.

```bash
mkdir agent-demo
cd agent-demo
npm init -y
npm pkg set type=module
npm install --save-exact langchain@1.5.11 @langchain/openai@1.5.12 better-sqlite3@12.11.1 zod@3.25.76
```

`type=module`은 아래 코드에서 `import`를 사용하기 위한 설정이다. 설치 후 만들어지는 `package-lock.json`도 보관하면 의존성 버전을 맞추는 데 도움이 된다.

## 1. 가상 직원 DB 만들기

`init-db.js`에 다음 내용을 작성한다. 동일한 파일을 다시 실행해도 샘플이 중복되지 않도록 기본키를 지정했다.

```javascript
import Database from "better-sqlite3";
import { fileURLToPath } from "node:url";

const db = new Database(fileURLToPath(new URL("./company.db", import.meta.url)));
try {
  db.exec(`
    CREATE TABLE IF NOT EXISTS employees (
      id INTEGER PRIMARY KEY,
      name TEXT NOT NULL,
      team TEXT NOT NULL,
      role TEXT NOT NULL
    )
  `);
  const insert = db.prepare(
    "INSERT OR IGNORE INTO employees (id, name, team, role) VALUES (?, ?, ?, ?)"
  );
  db.transaction(() => {
    insert.run(1, "김하늘", "개발팀", "백엔드 개발");
    insert.run(2, "이바다", "개발팀", "프론트엔드 개발");
    insert.run(3, "박나무", "분석팀", "데이터 분석");
  })();
} finally {
  db.close();
}
console.log("샘플 DB 준비 완료");
```

```bash
node init-db.js
```

## 2. DB와 날씨 함수를 Tool로 등록하기

`tools.js`에서는 DB 조회와 날씨 조회를 각각 하나의 Tool로 만든다. 날씨 예제는 도시명 변환 과정을 줄이기 위해 서울만 지원한다.

```javascript
import Database from "better-sqlite3";
import { fileURLToPath } from "node:url";
import { tool } from "langchain";
import { z } from "zod";

const dbPath = fileURLToPath(new URL("./company.db", import.meta.url));

export const getTeamMembers = tool(
  async ({ team }) => {
    const db = new Database(dbPath, { readonly: true });
    try {
      const members = db.prepare(
        "SELECT name, team, role FROM employees WHERE team = ? ORDER BY id"
      ).all(team);
      return JSON.stringify({ team, members });
    } finally {
      db.close();
    }
  },
  {
    name: "get_team_members",
    description: "팀 이름으로 가상 직원 DB에서 구성원과 담당 업무를 조회한다.",
    schema: z.object({
      team: z.string().min(1).describe("조회할 팀 이름. 예: 개발팀, 분석팀"),
    }),
  }
);

export const getWeather = tool(
  async ({ city }) => {
    if ((process.env.USE_MOCK_WEATHER ?? "true") === "true") {
      return JSON.stringify({
        city, temperature_c: 22, source: "mock",
        note: "테스트용 고정 응답이며 실제 날씨가 아닙니다.",
      });
    }

    const url = new URL("https://api.open-meteo.com/v1/forecast");
    url.search = new URLSearchParams({
      latitude: "37.5665",
      longitude: "126.9780",
      current: "temperature_2m",
      timezone: "Asia/Seoul",
    });

    try {
      const response = await fetch(url, {
        signal: AbortSignal.timeout(10000),
      });
      if (!response.ok) throw new Error(`HTTP ${response.status}`);
      const data = await response.json();
      if (!Number.isFinite(data.current?.temperature_2m)) {
        throw new Error("기온 데이터가 없습니다.");
      }
      return JSON.stringify({
        city,
        temperature_c: data.current.temperature_2m,
        time: data.current.time,
        source: "Open-Meteo",
      });
    } catch {
      return JSON.stringify({ city, error: "날씨 조회에 실패했습니다." });
    }
  },
  {
    name: "get_weather",
    description: "서울의 기온을 조회한다. 다른 도시는 지원하지 않는다.",
    schema: z.object({ city: z.enum(["서울"]) }),
  }
);

export const tools = [getTeamMembers, getWeather];
```

여기서 DB Tool의 입력은 SQL이 아니라 **팀 이름**이다. SQL은 개발자가 미리 정하고, 모델이 전달한 값은 `?` 파라미터로 바인딩한다. 이 예제에서 모델에 DB 전체를 수정하거나 임의의 쿼리를 실행할 기능은 주지 않았다.

Tool 설명도 중요하다. `get_team_members`의 이름만으로 부족한 부분을 설명과 스키마가 보완한다. 다만 스키마 검증은 입력 형식에 대한 검사이며, 실제 서비스의 사용자별 조회 권한을 대신하지는 않는다.

## 3. 모델과 Agent 연결하기

`.env`에는 사용할 모델 서버 정보를 넣는다. 아래 자리표시자는 실제 환경에 맞게 바꿔야 한다.

```dotenv
LLM_BASE_URL=http://127.0.0.1:8080/v1
LLM_MODEL=your-model-id
LLM_API_KEY=not-needed
USE_MOCK_WEATHER=true
```

위 주소는 로컬에서 실행한 호환 서버를 가정한 예시다. 서버에서 요구하는 인증 키가 있다면 `LLM_API_KEY`도 지정한다. 외부 모델 API를 사용한다면 해당 서비스의 주소·모델 ID·키로 변경한다.

이 예제는 `ChatOpenAI` 어댑터로 연결한다. **API 형식이 호환되는 것과 Tool 호출이 정상 동작하는 것은 별도로 확인해야 한다.** 로컬 모델을 사용한다면 모델과 서버의 Tool calling 지원 및 설정을 확인한다.

`.gitignore`도 작성한다.

```gitignore
node_modules/
.env
company.db
```

이어서 `agent.js`를 만든다.

```javascript
import { createAgent } from "langchain";
import { ChatOpenAI } from "@langchain/openai";
import { tools } from "./tools.js";

if (!process.env.LLM_BASE_URL || !process.env.LLM_MODEL) {
  throw new Error(".env에 LLM_BASE_URL과 LLM_MODEL을 설정하세요.");
}

const model = new ChatOpenAI({
  model: process.env.LLM_MODEL,
  apiKey: process.env.LLM_API_KEY || "not-needed",
  streamUsage: false,
  configuration: { baseURL: process.env.LLM_BASE_URL },
});

const agent = createAgent({
  model,
  tools,
  systemPrompt: [
    "한국어로 답하는 가상 직원 정보 도우미다.",
    "직원 정보는 반드시 DB Tool로, 날씨는 날씨 Tool로 확인한다.",
    "Tool 결과에 없는 사실을 만들지 않는다.",
    "조회 결과가 없거나 오류가 나면 그 사실을 알린다.",
    "날씨의 source가 mock이면 실제 날씨가 아닌 테스트 데이터임을 명시한다.",
    "서울 외 도시의 날씨는 지원하지 않는다고 답한다.",
  ].join(" "),
});

const question = process.argv.slice(2).join(" ")
  || "개발팀 구성원과 서울 날씨를 알려줘";

const result = await agent.invoke(
  { messages: [{ role: "user", content: question }] },
  { recursionLimit: 10 }
);

for (const message of result.messages) {
  for (const call of message.tool_calls ?? []) {
    console.log("TOOL CALL:", call.name, call.args);
  }
}
console.log("ANSWER:", result.messages.at(-1).content);
```

`recursionLimit`은 실행 단계가 끝없이 이어지는 상황을 제한하기 위한 값이다. 여기서 10은 Tool 호출 횟수나 API 비용의 상한을 뜻하지는 않는다.

## 4. 실행하면서 Tool 호출 확인하기

```bash
node --env-file=.env agent.js "개발팀 구성원과 서울 날씨를 알려줘"
```

아래는 동작을 설명하기 위한 **예상 출력 형태**다. 실제 모델 실행 로그를 옮긴 것은 아니며, 호출 순서와 답변 문장은 모델마다 달라질 수 있다.

```text
TOOL CALL: get_team_members { team: '개발팀' }
TOOL CALL: get_weather { city: '서울' }
ANSWER:
개발팀 구성원은 김하늘(백엔드 개발), 이바다(프론트엔드 개발)입니다.
서울 기온은 테스트 데이터 기준 22°C이며, 실제 날씨가 아닙니다.
```

한 번 답이 나왔다고 끝내기보다는 질문을 바꿔서 확인한다.

| 질문 | 확인할 동작 |
|---|---|
| 분석팀 구성원은 누구야? | DB 결과에 있는 박나무를 안내하는가 |
| 디자인팀 구성원을 알려줘 | 없는 직원을 만들지 않고 결과가 없다고 답하는가 |
| 서울 날씨 알려줘 | 날씨 Tool을 선택하고 mock임을 밝히는가 |
| 부산 날씨 알려줘 | 서울만 지원한다는 제한을 설명하는가 |

Tool 함수만 먼저 확인하고 싶다면 모델 서버 없이도 실행할 수 있다.

```bash
node --input-type=module -e 'import { getTeamMembers, getWeather } from "./tools.js"; console.log(await getTeamMembers.invoke({team:"개발팀"})); console.log(await getWeather.invoke({city:"서울"}));'
```

이 글의 코드에서는 가상 DB 조회, 조회 결과 없음, mock 날씨 응답을 로컬에서 확인했다. 실제 모델을 통한 Tool 선택과 외부 날씨 API 호출은 각 실행 환경에서 별도로 확인해야 한다.

## 5. 실제 날씨 API로 전환하기

`.env`의 값을 바꾸고 다시 실행한다.

```dotenv
USE_MOCK_WEATHER=false
```

이제 날씨 함수가 Open-Meteo에 요청을 보낸다. 예제에서 사용하는 값은 서울 좌표에 대한 `current.temperature_2m`이며, 응답에 제공되는 시각도 함께 반환한다. API의 사용 조건은 사용 목적에 맞게 확인한다. [Open-Meteo 문서](https://open-meteo.com/en/docs)

폐쇄망에서는 이 외부 주소에 직접 접근할 수 없으므로 mock으로 실행하거나, 접근 가능한 내부 API로 교체해야 한다. `USE_MOCK_WEATHER=true`는 날씨 호출만 대체한다. Agent를 실행하려면 모델 서버 연결은 여전히 필요하다.

## 이 예제로 알 수 있는 것

DB 조회와 HTTP 요청 자체는 일반 애플리케이션 개발에서도 하던 작업이다. 여기에 Tool의 설명과 입력 형식을 붙이면 모델이 사용할 수 있는 기능이 된다.

이번 구성에서는 직원 조회와 날씨 조회를 서로 독립적인 함수로 작성했다. 모델은 질문을 보고 필요한 기능을 선택하고, 실행부는 함수의 결과를 다시 모델에 전달한다. 실제 디버깅에서도 최종 답변만 보기보다는 **어떤 Tool에 어떤 인자가 들어갔고, 무엇이 반환됐는지**를 확인하는 것이 유용하다.

또한 DB를 조회한다고 해서 자동으로 벡터 검색 기반 RAG 구성이 되는 것은 아니다. 이 예제는 임베딩이나 벡터 DB 없이, 정해진 SQL을 실행하는 Tool을 Agent에 연결한 구성이다.

다음 단계에서는 이 구조를 바탕으로 사용할 업무 기능을 하나씩 추가할 수 있다. 먼저 작은 Tool을 단독으로 확인하고, 그다음 모델이 적절하게 선택하는지 확인하는 순서로 확장해보려 한다.
