# Blog Agent: 로컬 LLM 기반 자연어 블로그 관리 시스템

> 자연어 명령으로 게시글 CRUD를 수행하는 Function Calling 기반 AI Agent

`TypeScript` `OpenAI SDK` `LM Studio` `Function Calling` `Axios` `Session Auth`

## 1. 프로젝트 개요

### 배경

자연어로 API를 호출할 수 있는 시스템을 만들고 싶었습니다. 클라우드 LLM 대신 로컬에서 실행되는 LM Studio를 활용하여, 비용 부담 없이 Function Calling 기반의 AI Agent를 직접
구현해보는 것이 목표였습니다.

### 핵심 목표

- 자연어 인터페이스: "새 글 작성해줘", "3번 글 삭제해" 같은 명령 이해
- Function Calling 구현: LLM이 적절한 도구를 선택하고 실행
- 로컬 LLM 활용: LM Studio + Qwen3 8B로 외부 API 의존 없이 동작
- 기존 백엔드 연동: Spring API와 세션 기반 인증으로 통신

코드베이스: [github.com/kontang5/blog-agent](https://github.com/kontang5/blog-agent)

## 2. 아키텍처

```
┌─────────────────────────────────────────────────────────────────┐
│                           User (CLI)                            │
│                     "3번 게시글 제목 수정해줘"                       │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Main Loop                               │
│              readline · message history · tool call loop        │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                     LLM Client (OpenAI SDK)                     │
│               LM Studio 연결 · Function Calling 요청              │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      LM Studio (Qwen3 8B)                       │
│                   Tool selection · Argument extraction          │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                        Tool Executor                            │
│       create_post · get_posts · get_post · update_post · delete │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                    Spring Boot API Server                       │
│             Session Auth (Cookie) · Post CRUD Endpoints         │
└─────────────────────────────────────────────────────────────────┘
```

## 3. 주요 구현 내용

### 3.1 Tool Call Loop

LLM이 더 이상 도구를 호출하지 않을 때까지 반복 실행하는 핵심 루프입니다.

```typescript
// index.ts
async function processUserInput(input: string): Promise<string> {
    messages.push({role: "user", content: input});

    let completion = await chat(messages, toolDefinitions);
    let assistantMessage = completion.choices[0].message;

    // Tool call loop - LLM이 tool_calls를 반환하는 동안 계속 실행
    while (assistantMessage.tool_calls && assistantMessage.tool_calls.length > 0) {
        messages.push(assistantMessage);

        for (const toolCall of assistantMessage.tool_calls) {
            const args = JSON.parse(toolCall.function.arguments);
            const result = await executeTool(toolCall.function.name, args);

            messages.push({
                role: "tool",
                tool_call_id: toolCall.id,
                content: JSON.stringify(result),
            });
        }

        // 도구 실행 결과를 포함하여 다시 LLM에 요청
        completion = await chat(messages, toolDefinitions);
        assistantMessage = completion.choices[0].message;
    }

    return assistantMessage.content || "";
}
```

| 단계                 | 역할                           |
|--------------------|------------------------------|
| 1. User Input      | 메시지 히스토리에 사용자 입력 추가          |
| 2. LLM Request     | Tool definitions와 함께 LLM에 요청 |
| 3. Tool Execution  | 반환된 tool_calls를 순회하며 실행      |
| 4. Result Feedback | 실행 결과를 메시지에 추가하고 재요청         |
| 5. Final Response  | tool_calls가 없으면 최종 응답 반환     |

> 다중 도구 호출과 연쇄 호출을 모두 지원하는 구조

### 3.2 OpenAI 호환 Tool Definitions

OpenAI Function Calling 형식으로 도구를 정의하여 LM Studio와 호환됩니다.

```typescript
// post-tools.ts
export const toolDefinitions: ToolDefinition[] = [
    {
        type: "function",
        function: {
            name: "create_post",
            description: "Create a new blog post",
            parameters: {
                type: "object",
                properties: {
                    title: {type: "string", description: "The title of the post"},
                    content: {type: "string", description: "The content of the post"},
                },
                required: ["title", "content"],
            },
        },
    },
    // get_posts, get_post, update_post, delete_post ...
];
```

| Tool        | Parameters         | 설명           |
|-------------|--------------------|--------------|
| create_post | title, content     | 새 게시글 생성     |
| get_posts   | -                  | 전체 게시글 목록 조회 |
| get_post    | id                 | 특정 게시글 조회    |
| update_post | id, title, content | 게시글 수정       |
| delete_post | id                 | 게시글 삭제       |

### 3.3 Tool Executor와 에러 처리

도구 이름을 기반으로 적절한 API 함수를 호출하고, 결과를 표준화된 형식으로 반환합니다.

```typescript
// post-tools.ts
export async function executeTool(name: string, args: Record<string, unknown>): Promise<ToolResult> {
    try {
        switch (name) {
            case "create_post":
                const created = await postApi.createPost({
                    title: args.title as string,
                    content: args.content as string,
                });
                return {success: true, data: created};

            case "delete_post":
                await postApi.deletePost(args.id as number);
                return {success: true, data: {message: "Post deleted successfully"}};

            default:
                return {success: false, error: `Unknown tool: ${name}`};
        }
    } catch (error) {
        return {success: false, error: error instanceof Error ? error.message : String(error)};
    }
}
```

> try-catch로 API 오류를 잡아 LLM에게 실패 정보를 전달, 재시도나 대안 제시 가능

### 3.4 세션 기반 자동 인증

Cookie Jar를 활용해 세션을 유지하고, 필요시 자동으로 로그인합니다.

```typescript
// post-api.ts
import {wrapper} from "axios-cookiejar-support";
import {CookieJar} from "tough-cookie";

const jar = new CookieJar();
const api = wrapper(axios.create({
    baseURL: config.springApi.baseUrl,
    jar,  // 쿠키 자동 관리
}));

let isLoggedIn = false;

async function ensureLoggedIn(): Promise<void> {
    if (!isLoggedIn) {
        await login(config.springApi.username, config.springApi.password);
    }
}

export async function createPost(req: CreatePostRequest): Promise<Post> {
    await ensureLoggedIn();  // 모든 API 호출 전 인증 확인
    const {data} = await api.post<Post>("/v1/posts", req);
    return data;
}
```

| 구성 요소                   | 역할                         |
|-------------------------|----------------------------|
| CookieJar               | 서버에서 발급한 세션 쿠키 저장          |
| axios-cookiejar-support | axios 요청에 쿠키 자동 첨부         |
| ensureLoggedIn()        | Lazy 인증 - 첫 API 호출 시에만 로그인 |

> 세션 쿠키를 사용하여 Spring Security의 기본 인증 방식과 호환

### 3.5 LM Studio 연동

OpenAI SDK를 그대로 사용하면서 baseURL만 변경하여 로컬 LLM에 연결합니다.

```typescript
// llm-client.ts
const client = new OpenAI({
    baseURL: config.lmStudio.baseUrl,  // http://localhost:1234/v1
    apiKey: "lm-studio",
});

export async function chat(
    messages: OpenAI.ChatCompletionMessageParam[],
    tools: ToolDefinition[]
): Promise<OpenAI.ChatCompletion> {
    const completion = await client.chat.completions.create({
        model: config.lmStudio.model,
        messages,
        tools,  // Function calling definitions
    });
    return completion;
}
```

> OpenAI 호환 API를 제공하는 서비스로 교체 가능

## 4. 사용 예시

```
Blog Agent CLI (type 'exit' to quit)

You: 새 글 작성해줘. 제목은 "첫 번째 포스트", 내용은 "안녕하세요!"

[Tool] create_post({"title":"첫 번째 포스트","content":"안녕하세요!"})
[Result] {"success":true,"data":{"id":1,"title":"첫 번째 포스트",...}}