
# MCP TypeScript SDK

https://www.npmjs.com/package/@modelcontextprotocol/sdk

LLM(Large Language Model)과의 상호 작용을 위해 설계된 프로토콜입니다. 이 SDK를 활용하면 LLM 애플리케이션에 \*\*데이터(Resources)\*\*와 \*\*기능(Tools)\*\*을 노출하는 서버, 즉 플러그인을 쉽게 구축할 수 있습니다.

## 💡 핵심 개념

MCP는 LLM이 외부 세계와 소통하는 방식을 표준화합니다. 주요 핵심 개념은 다음과 같습니다.

  * **Resources (리소스)**: LLM에 데이터를 노출합니다. 리소스는 데이터를 제공하지만, 중요한 계산을 수행하거나 `side effect`를 발생시키지 않아야 합니다.
  * **Tools (도구)**: LLM에 특정 기능을 제공합니다. 리소스와 달리 연산을 수행하거나 `side effect`를 가질 수 있습니다.
  * **Prompts (프롬프트)**: LLM과 상호 작용하기 위한 패턴을 정의하는 재사용 가능한 템플릿입니다.

-----

## 🚀 `McpServer`

`McpServer`는 MCP의 핵심 인터페이스입니다. 이 클래스는 연결 관리, 프로토콜 준수 및 메시지 라우팅을 담당합니다.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server";

const server = new McpServer({
  name: "my-app", // 서버의 고유 이름
  version: "1.0.0" // 서버의 버전
});
```

-----

## 📚 Resources 등록하기

LLM이 URI를 통해 접근할 수 있는 데이터를 정의합니다. `registerResource`는 다음과 같은 파라미터를 가집니다.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server";

// ... (McpServer 인스턴스 생성)

server.registerResource(
  "config", // 고유한 문자열 식별자 (리소스의 이름)
  "config://app", // 리소스의 URI 패턴을 정의하는 문자열 또는 ResourceTemplate 객체
  /**
   * 리소스에 대한 메타데이터를 포함하는 객체입니다.
   */
  {
    title: "Application Config", // 리소스의 사용자 친화적인 표시 이름
    description: "Application configuration data", // 리소스의 기능에 대한 간략한 설명
    mimeType: "text/plain" // (선택 사항) 리소스 콘텐츠의 MIME 유형
  },
  /**
   * URI 및 추출된 파라미터를 받아 리소스 콘텐츠를 반환하는 비동기 함수입니다.
   * 이 함수는 `contents` 속성을 포함하는 객체를 반환해야 합니다.
   */
  async (uri) => ({
    contents: [{
      uri: uri.href,
      text: "App configuration here"
    }]
  })
);

// 파라미터가 동적인 리소스
server.registerResource(
  "user-profile",
  new ResourceTemplate("users://{userId}/profile", { list: undefined }),
  {
    title: "User Profile",
    description: "User profile information"
  },
  async (uri, { userId }) => ({
    contents: [{
      uri: uri.href,
      text: `Profile data for user ${userId}`
    }]
  })
);
```

MCP 서버 인스턴스가 생성된 후 서버의 초기 설정 단계에서 호출되어 해당 리소스 정의가 서버 내부적으로 저장된다.

- LLM 은 대화 맥락에서 특정 정보를 필요로 할 때, 서버에 등록된 리소스 목록 (title, description, URI 템플릿 등)을 참조할 수 있다.
- 즉, LLM은 `registerResource` 로 등록한 title, description을 참조하여 어떤 리소스가 필요한지를 판단한다.

> tool 또한 동일하게 동작한다

-----

## 🛠️ Tools 등록하기

LLM이 서버를 통해 특정 작업을 수행할 수 있도록 도구를 정의합니다. `registerTool`은 다음과 같은 파라미터를 가집니다.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server";
import { z } from "zod"; // Zod는 스키마 정의를 위해 사용됩니다.

// ... (McpServer 인스턴스 생성)

server.registerTool(
  "calculate-bmi", // 도구의 고유한 문자열 식별자 (도구의 이름)
  /**
   * 도구에 대한 메타데이터와 스키마를 포함하는 객체입니다.
   */
  {
    title: "BMI Calculator", // 도구의 사용자 친화적인 표시 이름
    description: "Calculate Body Mass Index", // 도구 기능에 대한 간략한 설명
    inputSchema: z.object({ // 도구가 예상하는 입력 매개변수를 정의하는 Zod 스키마
      weightKg: z.number(), // 체중 (kg)
      heightM: z.number() // 키 (m)
    })
  },
  /**
   * 입력 파라미터를 비동기적으로 처리하고 결과를 반환하는 함수입니다.
   * 이 함수는 `inputSchema`에 따라 유효성 검사를 거친 파라미터를 받고, `content` 속성을 포함하는 객체를 반환해야 합니다.
   */
  async ({ weightKg, heightM }) => ({
    content: [{
      type: "text",
      text: String(weightKg / (heightM * heightM))
    }]
  })
);
```

-----

## 💬 Prompts 등록하기

LLM이 서버와 효과적으로 상호작용하는 데 도움이 되는 재사용 가능한 프롬프트 템플릿을 정의합니다. `registerPrompt`는 다음과 같은 파라미터를 가집니다.

```typescript
import { McpServer } from "@modelcontextprotocol/sdk/server";
import { completable } from "@modelcontextprotocol/sdk/server/completable.js";
import { z } from "zod"; // Zod는 스키마 정의를 위해 사용됩니다.

// ... (McpServer 인스턴스 생성)

server.registerPrompt(
  "review-code", // 프롬프트의 고유한 문자열 식별자 (프롬프트의 이름)
  /**
   * 프롬프트에 대한 메타데이터와 스키마를 포함하는 객체입니다.
   */
  {
    title: "Code Review", // 프롬프트의 사용자 친화적인 표시 이름
    description: "Review code for best practices and potential issues", // 프롬프트에 대한 간략한 설명
    argsSchema: z.object({ code: z.string() }) // 프롬프트가 예상하는 인수를 정의하는 Zod 스키마
  },
  /**
   * 프롬프트 파라미터를 받아 LLM에 전달할 프롬프트 메시지를 생성하는 함수입니다.
   * 이 함수는 `messages` 속성을 포함하는 객체를 반환해야 합니다.
   */
  ({ code }) => ({
    messages: [{
      role: "user",
      content: {
        type: "text",
        text: `Please review this code:\n\n${code}`
      }
    }]
  })
);
```

MCP 서버 인스턴스가 생성된 후 서버의 초기 설정 단계에서 호출되어 해당 프롬프트를 정의하고 등록한다.

- 직접 LLM에 메시지를 보내는 대신, Prompt를 통해 정의된 템플릿과 로직을 사용하여 LLM에게 징의를 생성하고 전달할 수 있다.
- LLM은 title과 description를 참조하여 어떤 프롬프트가 사용자의 의도에 적합한지 판단하고 결정한다.
- 사용자와 프롬프트의 텍스트를 표준화하는 템플릿이라고 보면 됨

-----

## 🌐 서버 실행 및 전송 방식

MCP 서버는 클라이언트와 통신하기 위해 `Transport`와 연결되어야 합니다. 서버를 시작하는 방법은 어떤 `Transport`를 사용하는지에 따라 달라집니다.

### 전송 방식의 종류

MCP는 다양한 환경에서 유연하게 작동할 수 있도록 여러 전송 방식을 지원합니다.

  * **StdIO (Standard Input/Output)**:

      * 주로 로컬 프로세스 간 통신에 사용됩니다. (예: 데스크톱 애플리케이션)
      * 클라이언트가 MCP 서버를 자식 프로세스로 시작하고, `stdin`으로 메시지를 전송하며 `stdout`에서 응답을 읽는 방식으로 작동합니다.
      * JSON-RPC 메시지를 줄 단위로 교환합니다.

  * **HTTP (Hypertext Transfer Protocol)**:

      * 원격 서버와의 통신에 사용됩니다. (예: 클라우드 서비스 연동)
      * RESTful API를 통한 원격 접근을 지원합니다.

  * **SSE (Server-Sent Events)**:

      * 실시간 알림 수신에 사용됩니다. (예: 라이브 데이터 스트리밍)
      * 클라이언트가 HTTP 연결을 통해 서버에 연결하면, 서버는 `text/event-stream` 콘텐츠 타입으로 응답합니다.
      * 클라이언트가 별도의 HTTP 요청으로 메시지를 전송하고, 서버는 스트림을 통해 이벤트 형태로 응답을 전송합니다.

-----