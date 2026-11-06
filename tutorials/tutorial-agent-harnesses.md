# How an Agent Harness Works

*From one message to a working agent, explained three times, each time in more detail*

This tutorial explains how an AI agent harness works, starting from a single message sent to a chat model. It tells the same story three times: first in one paragraph, then in five, then piece by piece with real examples of what goes over the wire. The first two passes give you the shape of the whole thing. The third is for when you need to know how each part actually works.

## In one paragraph

A language model takes text in and sends text out, and it remembers nothing between calls. To hold a conversation, your program keeps a list of messages and sends the whole list every time, adding each reply and each new question to the end, so every request is bigger than the last. The first entry in that list is the system prompt, where the developer tells the model who it is and how to behave. To let the model act, the program also sends a catalog of tools, each described by a name, a description, and a schema for its arguments. The model can't run anything itself. When it wants a tool, it replies with a structured request, and the program runs the tool, appends the result to the list, and calls the model again. MCP is a standard protocol for plugging tools from outside programs into that catalog. Repeating call, run, append until the model answers in plain text is the tool loop. Because the list grows with every step, the program eventually has to shrink it, usually by replacing old history with a summary, and that is compaction. The program that does all of this is the harness: everything around the model that turns a text predictor into an agent.

## In five paragraphs

Start with a single exchange. Your program sends an HTTP request with a list of messages, each tagged with a role such as user or assistant. On the server, a chat template flattens that list into one long stream of tokens, which are chunks of text a few characters long, with special markers between the turns. The model reads the stream and predicts a reply one token at a time until it produces an end marker. Then it forgets everything. So for the second message, your program sends the first question, the first answer, and the new question together. Every request contains the whole conversation so far, which means each request is bigger than the last, and the total amount of text processed grows much faster than the conversation does. Providers soften the cost with prompt caching, which reuses the work already done on any prefix that exactly matches a recent request.

The system prompt is the first message in the list. It comes from the developer rather than the user, and models are trained to give it more authority than anything that follows, while giving text from tools and web pages no authority at all. In an agent, the system prompt sets the identity, the tone, the safety rules, and the way of working: plan, use the tools, check the result, and report back. Harnesses keep it fixed so it stays cached, and they put the things that change, such as the date, the working folder, and project rules files like AGENTS.md, in messages right after it.

Tools are how a model acts on the world. The harness sends a catalog of tool definitions with every request, and each definition is a name, a plain-language description, and a JSON Schema for the arguments. The provider renders the catalog into the prompt as text, so it costs tokens on every call, and a model given too many tools starts picking the wrong ones. When the model decides a tool would help, it replies with a tool call instead of an answer: the tool's name, its arguments, and an ID. The harness checks the call, runs the tool, and appends the output as a tool result tagged with the same ID. MCP, the Model Context Protocol, standardizes where tools come from. The harness connects to MCP servers, asks each one for its tools, merges them into the catalog, and forwards the model's calls to whichever server owns the tool.

One tool call rarely finishes a task, so the harness runs a loop. It calls the model, runs whatever tools the model asked for, appends the results, and calls the model again, until the model replies with plain text and no tool calls. A coding agent fixing a bug goes around this loop several times: run the tests, read the file, edit it, run the tests again, and report. The loop is also where the harness keeps the agent under control. It caps the number of rounds, asks the user before risky actions, runs commands in a sandbox, runs independent calls in parallel, streams progress to the screen, and lets the user interrupt.

All that looping makes the list long. Every file read and every test log stays in it, and eventually the list approaches the context window, the most text the model can take in at once. Quality drops well before that point, because models get worse at using information as their input grows. So the harness compacts. It clears out old tool outputs, or it asks the model to write a handoff summary of the work so far and replaces the old history with that summary plus the most recent messages. Put the pieces together (the message list, the system prompt, the tool catalog, tool execution, MCP connections, the loop and its guardrails, and compaction) and you have a harness. The model supplies the judgment. The harness supplies everything else.

## Piece by piece

Here is the same story again, one piece at a time.

### 1. One message, one answer

When you type into a chat app and press Enter, the app sends an HTTP POST request to the model provider. The body is JSON naming a model and a list called `messages`, where each message has a `role` and some `content`. Here is a minimal request in the shape of OpenAI's Chat Completions API. Most of the industry copies this format, so when a server or tool says it's "OpenAI-compatible," this is what it means.

```json
{
  "model": "your-model",
  "messages": [
    { "role": "user", "content": "What is a token?" }
  ]
}
```

And here is the shape of what comes back:

```json
{
  "choices": [{
    "message": { "role": "assistant", "content": "A token is a small chunk of text, often part of a word..." },
    "finish_reason": "stop"
  }],
  "usage": { "prompt_tokens": 12, "completion_tokens": 48, "total_tokens": 60 }
}
```

Two fields matter besides the reply itself. `finish_reason` says why the model stopped: `stop` means it finished naturally, `length` means it ran out of room, and `tool_calls` means it wants to use a tool (section 5). `usage` says how many tokens went in and came out, which is what you pay for. The prompt count is higher than the handful of words you typed, because OpenAI also counts "formatting tokens used for request structure, such as message roles and boundaries." That formatting is the next thing to look at.

Every major provider uses the same idea with different field names:

| | OpenAI Chat Completions | Anthropic Messages | Google Gemini |
|---|---|---|---|
| Endpoint | `POST /v1/chat/completions` | `POST /v1/messages` | `generateContent` (Google now recommends its Interactions API) |
| The list | `messages` | `messages` | `contents` |
| Roles | `developer` or `system`, `user`, `assistant`, `tool` | `user`, `assistant` | `user`, `model` |
| Why it stopped | `finish_reason` | `stop_reason` | `finishReason` |
| Token counts | `usage` | `usage` | `usageMetadata` |

OpenAI also has a newer Responses API, which takes `input` and `instructions` instead of `messages`. OpenAI recommends it for new projects, and its newest models only support tool calling there. The ideas in this tutorial work the same way in every format.

**What the server does with your request.** The model never sees JSON. The server first runs your messages through a chat template, which flattens the list into one stream of text with special marker tokens between the turns. Then a tokenizer chops that text into tokens, which average about four characters of English text (OpenAI's rule of thumb is that 100 tokens is about 75 words). Here is what the server builds for a one-question conversation, in OpenAI's harmony format, which is documented for its open-weight gpt-oss models:

```text
<|start|>user<|message|>What is 2 + 2?<|end|>
<|start|>assistant
```

The template ends by opening an assistant turn and leaving it unfinished. The model's only real skill is predicting the next token, so it continues the pattern. It writes the assistant's reply one token at a time and stops when it produces a stop token. Roles, turns, and the whole idea of "chat" are conventions the model learned in training, sitting on top of plain next-token prediction. As Hugging Face's documentation puts it, "All causal LMs, whether chat-trained or not, continue a sequence of tokens." Hosted providers like OpenAI and Anthropic don't publish their templates, but they work the same way.

**Prefill and decode.** Generation happens in two phases. In prefill, the model processes the whole prompt in parallel. That's fast per token, and it builds the KV cache, the model's working memory of everything it has read. In decode, the model generates the reply one token at a time, and each new token depends on all the ones before it, so decode is much slower per token. Time to first token is mostly waiting in the queue plus prefill, so longer prompts take longer to start answering. Most apps also turn on streaming, where the server sends each small piece of the reply as soon as it exists, as server-sent events. That's why chat replies appear word by word.

Three things to take away from a single exchange:

- The model sees one flat stream of tokens, not structured messages. The structure is a convention.
- It writes one token at a time, and the stop reason tells you why it stopped.
- When the response is finished, the model keeps nothing. No memory of the exchange survives on its side.

### 2. The second message, and the list that keeps growing

Now the user asks a follow-up. The model has no memory of the first exchange, and Anthropic's documentation says so plainly: "The Messages API is stateless, which means that you always send the full conversational history." So the app appends the model's reply and the new question to the list, and sends all of it:

```json
{
  "model": "your-model",
  "messages": [
    { "role": "user", "content": "What is a token?" },
    { "role": "assistant", "content": "A token is a small chunk of text, often part of a word..." },
    { "role": "user", "content": "How many tokens is a typical page?" }
  ]
}
```

At the token level, here is the harmony example from section 1 after one more turn. The model reads the whole history and continues from the end of it:

```text
<|start|>user<|message|>What is 2 + 2?<|end|>
<|start|>assistant<|channel|>final<|message|>2 + 2 = 4.<|end|>
<|start|>user<|message|>What about 9 / 2?<|end|>
<|start|>assistant
```

From the model's point of view, there is no "second message." There is only a longer document to continue. This has a useful consequence: the history is whatever your program says it is. If the app edits or deletes an old message before the next request, the model has no way to know it was ever different. Harnesses rely on this all the time, and compaction (section 8) is built on it.

**The cost of resending everything.** Because every request includes everything before it, each one is bigger than the last. Here is a simple conversation with a 500-token system prompt, 50-token user messages, and 300-token replies:

| Turn | Input tokens in this request | Input tokens sent in total so far |
|---|---|---|
| 1 | 550 | 550 |
| 2 | 900 | 1,450 |
| 5 | 1,950 | 6,250 |
| 10 | 3,700 | 21,250 |
| 20 | 7,200 | 77,500 |
| 50 | 17,700 | 456,250 |

Each request grows in a straight line, but the running total grows roughly with the square of the number of turns. By turn 20 you've sent about seven times as many tokens as twenty copies of the first request would be. Agents make this much worse, because a single tool result, such as a file's contents or a test log, can be thousands of tokens, and it stays in the list for every request after it.

**Prompt caching.** Providers soften this with prompt caching. When a request starts with exactly the same tokens as a recent one, the server can reuse the prefill work it already did instead of recomputing it, and it charges much less for those tokens. The important word is exactly. The cache covers the longest identical prefix, so one changed character ends the match, and everything after that point is processed and billed at full price. Providers build the prefix in a fixed order: tool definitions first, then the system prompt, then the messages. Anthropic's docs spell out the consequence: "Changes at each level invalidate that level and all subsequent levels." That's why well-built harnesses treat the front of the request as fixed and only ever append to the end.

Here's how caching looks at the three big providers as of November 2026. Prices change often, so treat the numbers as a snapshot:

| | OpenAI | Anthropic | Google Gemini |
|---|---|---|---|
| How it turns on | Automatic | Automatic, or `cache_control` breakpoints you place | Implicit caching, on by default |
| Shortest prefix it will cache | 1,024 tokens | 512 tokens on Claude 5.x models | 4,096 tokens on Gemini 4 Argon |
| Price of a cached read | 10% of normal input, 5% on some models | 10% of normal input, 5% on Opus 5.5 and Sonnet 5.5 | discounted, with "no cost saving guarantee" |
| How long a cache entry lives | 30 minutes by default, or 12 hours | 5 minutes or 1 hour | not stated |

**Letting the server keep the history.** Some APIs offer to store the conversation for you. OpenAI's Responses API can chain requests with `previous_response_id`, its Conversations API keeps a durable conversation object, Google's Interactions API keeps state on the server, and Anthropic launched a message threads beta in November 2026. These save your program from resending the history over the network, but they don't change what the model does. Anthropic's announcement is direct about it: "Claude still reads the full conversation on every turn, so token usage stays the same." OpenAI likewise bills all previous input tokens in a chain as input tokens. Statelessness belongs to the model, not to the API.

**The ceiling.** Every model has a context window, the most tokens it can take in at once. On some models the window is shared between input and output. Flagship models in late 2026 have windows of around one million tokens, which sounds enormous. But an agent that spends hours reading files and logs can still fill it, and quality starts slipping long before it's full. Section 8 deals with that.

### 3. The system prompt

So far every message came from either the user or the model. The system prompt comes from a third party: the developer who built the app. It sits at the very start of the conversation and tells the model who it is, what it's for, and what rules to follow. Each API has its own place for it:

| API | Where the system prompt goes |
|---|---|
| OpenAI Chat Completions | A message with role `developer`, which replaced `system` starting with the o1 models |
| OpenAI Responses | The top-level `instructions` field, or developer messages in `input` |
| Anthropic Messages | The top-level `system` field, kept apart from `messages`. Claude 5 models also accept system messages later in the conversation. |
| Google Gemini | The top-level `systemInstruction` field |

Under the hood, it's still just tokens. The chat template renders it first, under its own role marker, and the model was trained to treat text under that marker as instructions from whoever deployed it. Here is a shortened harmony example. Harmony splits the job in two: the `system` section holds identity and dates, and the `developer` section holds what most people call the system prompt, plus the tool definitions.

```text
<|start|>system<|message|>You are ChatGPT, a large language model trained by OpenAI.
Knowledge cutoff: 2024-06
Current date: 2025-06-28<|end|>
<|start|>developer<|message|># Instructions

Use a friendly tone.<|end|>
<|start|>user<|message|>What is the weather like in SF?<|end|>
<|start|>assistant
```

**Where its authority comes from.** What makes the system prompt special is authority, and that authority is trained into the model, not enforced by code. A 2024 OpenAI paper, "The Instruction Hierarchy," found that models were treating system prompts with "the same priority as text from untrusted users and third parties." It then showed that training a ranking into the model made it much harder to override. OpenAI's Model Spec now spells out a full chain of command, and Anthropic's guidance draws the same line around tool output, saying instructions found inside tool results are "information rather than ... commands." The result looks like this:

| Authority | Who is speaking | Example |
|---|---|---|
| Highest | The model's maker | Built-in safety policies |
| High | The developer, through the system prompt | "You are a coding agent. Never push to main." |
| Medium | The user | "Fix the failing test in auth.rs." |
| None by default | Tool outputs, files, web pages, quoted text | A README that says "ignore your previous instructions" |

That bottom row matters a lot for agents, because almost everything an agent reads comes in through tools.

**What's inside an agent's system prompt.** Agent harnesses have long system prompts, usually a few thousand tokens. You can read real ones: OpenAI's Codex CLI and Google's Gemini CLI are open source, and Anthropic publishes the prompts it uses in its Claude apps. They share a common shape:

- **Identity.** "You are a coding agent running in the Codex CLI..." or "You are Gemini CLI, an interactive CLI agent..."
- **Tone and formatting.** How long answers should be, when to use markdown, and how to write progress updates.
- **How to work.** Make a plan, keep going until the task is actually done, check your work, and summarize at the end.
- **Tool rules.** Prefer the dedicated file tools over shell commands, and run independent calls in parallel.
- **Safety rules.** Don't commit or push unless asked, be careful with destructive commands, and never expose secrets.
- **Trust rules.** Treat tool output as data, not instructions. Gemini CLI wraps tool output in `<untrusted_context>` tags for exactly this reason.
- **Conditional sections.** Extra text that only appears in plan mode, inside a sandbox, or when the folder is a git repository.

**What's deliberately left out.** Notice what's missing from that list: today's date, the current folder, the git status, and the project's own rules. Those things change, and anything that changes inside the system prompt breaks the prompt cache for everything after it. Anthropic's caching docs describe the classic mistake. With a timestamp in the cached block, "You pay for a fresh cache write on every request and never get a read." So harnesses keep the system prompt static and put the changing details in the first messages after it:

| Harness | System prompt | Project rules file | Date and environment |
|---|---|---|---|
| Claude Code | `system` field | `CLAUDE.md`, sent as a user message after the system prompt | Sent in the conversation |
| Codex CLI | `instructions` | `AGENTS.md`, sent as a user-role message in `<INSTRUCTIONS>` tags (capped at 32 KiB) | An `<environment_context>` message, resent only when something changes |
| Gemini CLI | `systemInstruction` | Global `GEMINI.md` in the system prompt, project files in the first user message | A `<session_context>` block in the first user message |

Project rules files deserve a word of their own. AGENTS.md, which its site calls "a README for agents," is used by over 60,000 open-source projects and is now looked after by the Agentic AI Foundation under the Linux Foundation. Harnesses look for it, or their own equivalent such as CLAUDE.md, GEMINI.md, or Cursor's rules files, and put it near the top of the conversation. When there are several, the one closest to the file being edited wins.

**It is not a security boundary.** Users can often talk a model into revealing its system prompt, and text injected through a tool can sometimes override it. OWASP's guidance is blunt: "the system prompt should not be considered a secret, nor should it be used as a security control." Anything that really must be enforced, such as which folders a tool may write to, has to live in the harness code, where the model can't argue with it.

### 4. Offering tools: the tool catalog

On its own, a model can only produce text. Tools let it act: read a file, run a command, search the web. To offer a tool, the harness adds a definition of it to every request. A definition has three parts: a name, a description in plain language, and a JSON Schema describing the arguments. Here is one in OpenAI's Chat Completions format:

```json
{
  "type": "function",
  "function": {
    "name": "read_file",
    "description": "Read a text file from the project and return its contents. Use this before editing a file.",
    "parameters": {
      "type": "object",
      "properties": {
        "path": { "type": "string", "description": "Path relative to the project root, e.g. src/main.rs" }
      },
      "required": ["path"]
    }
  }
}
```

Anthropic's format is nearly identical, with the schema in a field called `input_schema`. Gemini wraps a list of definitions in `functionDeclarations`. Names are short identifiers: OpenAI's Chat Completions allows letters, digits, underscores, and dashes up to 64 characters, and Anthropic allows up to 128. The full list of definitions sent with a request is the tool catalog.

**How the catalog reaches the model.** The definitions don't enter the model through some separate channel. OpenAI's documentation explains: "Under the hood, functions are injected into the system message in a syntax the model has been trained on. This means callable function definitions count against the model's context limit and are billed as input tokens." Open models make this easy to see, because their chat templates are public. Here is what Qwen3's template produces for a weather tool:

```text
<|im_start|>system
You are a helpful assistant.

# Tools

You may call one or more functions to assist with the user query.

You are provided with function signatures within <tools></tools> XML tags:
<tools>
{"type": "function", "function": {"name": "get_weather", "description": "Get the current weather for a city.", "parameters": {"type": "object", "properties": {"city": {"type": "string", "description": "City name, e.g. Paris"}}, "required": ["city"]}}}
</tools>

For each function call, return a json object with function name and arguments within <tool_call></tool_call> XML tags:
<tool_call>
{"name": <function-name>, "arguments": <args-json-object>}
</tool_call><|im_end|>
<|im_start|>user
What is the weather in Paris?<|im_end|>
<|im_start|>assistant
```

So the catalog is really part of the prompt: text at the top that explains what's available and exactly how to ask for it. Anthropic does the same thing and documents the first line of its hidden tool prompt: "In this environment you have access to a set of tools you can use to answer the user's question." Other model families use other layouts. Llama 3.1 was trained with tools in the user turn, and gpt-oss sees them as TypeScript-style type declarations. Whatever the layout, the model learned during training to read it and to answer in the matching format.

**What the catalog costs.** Because the catalog is prompt text, you pay for it on every request, and with many tools it adds up fast. Anthropic adds a hidden tool-use prompt of a few hundred tokens whenever tools are present (286 tokens on its 5.5 models), and the definitions are billed on top of that. In one of Anthropic's examples, 58 tools from five MCP servers took about 55,000 tokens "before the conversation even starts." Accuracy suffers too. Anthropic reports that tool selection "degrades once you exceed 30-50 available tools," and OpenAI suggests keeping fewer than 20 available at the start of a turn.

**Steering tool use.** Every API lets you control whether the model may call tools on a given request:

| Setting | OpenAI | Anthropic | Gemini | Effect |
|---|---|---|---|---|
| Model decides | `auto` | `auto` | `AUTO` | The default |
| Must call some tool | `required` | `any` | `ANY` | Forces a tool call |
| Must call one specific tool | a named function | `tool`, with a name | `ANY`, plus `allowed_function_names` | Forces that tool |
| No tool calls | `none` | `none` | `NONE` | Tools stay listed but can't be called |

Two wrinkles are worth knowing. OpenAI also has `allowed_tools`, which limits a turn to a subset of the catalog without changing the catalog itself, so the prompt cache survives. And Anthropic's newest models no longer accept the two forcing options at all.

**Writing tools the model can use well.** The model picks tools only by reading their names, descriptions, and schemas, so writing a tool is a lot like writing documentation for a new hire. The advice in Anthropic's "Writing effective tools for agents" and OpenAI's function-calling guide comes down to a short list:

- **Write real descriptions.** Anthropic calls the description "by far the most important factor in tool performance" and asks for at least three or four sentences.
- **Build fewer, bigger tools.** One `schedule_event` tool works better than separate tools to list free slots and create events.
- **Namespace the names.** Prefixes such as `asana_projects_search` keep similar tools from different services apart.
- **Make bad calls impossible.** Use enums and required fields, and don't ask the model for values your code already knows.
- **Return useful, compact results.** Use readable names instead of IDs, page or truncate long outputs, and say what to do next. In Anthropic's Slack example, a concise response took 72 tokens instead of 206.

**Very large catalogs.** When a harness has hundreds of tools, it stops sending them all. Tool search sends a search tool, and maybe a list of names, up front, then loads full definitions only when the model asks for them. Anthropic's version marks tools with `defer_loading` and reports about 85% fewer tokens, with accuracy on its MCP evaluation rising from 49% to 74% for one model. OpenAI added a similar `tool_search` in March 2026, and Claude Code, VS Code, and Cursor all defer MCP tools in their own ways.

**Tools that run somewhere else.** One last distinction. Some tools run on the provider's own servers. Web search, code execution, and similar built-in tools are executed by the provider within a single API call, and your code never handles them. Everything in the rest of this tutorial is about the other kind, client tools, where the model asks and your harness does the work.

### 5. Tool calls and tool results

Here is the single most important fact about tool calling: the model never runs anything. Google's documentation puts it in one line: "The Model doesn't execute the function itself." When the model decides a tool would help, it writes a request for one, and the response comes back with a different stop reason. In Chat Completions, the reply has `finish_reason: "tool_calls"` and looks like this:

```json
{
  "role": "assistant",
  "content": null,
  "tool_calls": [{
    "id": "call_abc123",
    "type": "function",
    "function": { "name": "read_file", "arguments": "{\"path\": \"src/auth.rs\"}" }
  }]
}
```

Notice that the arguments arrive as a JSON string, not a JSON object, so your code has to parse them. OpenAI warns that the model "does not always generate valid JSON, and may hallucinate parameters," so that parse can fail.

**What the harness does with a call.** At this point the harness takes over. In order, it:

1. Parses the arguments.
2. Checks that the tool exists and that the arguments fit its schema.
3. Checks permission: is the agent allowed to do this, and does the user need to approve it first?
4. Runs the tool, with a timeout.
5. Trims the output if it's huge.
6. Appends the output as a tool result tagged with the same ID, and calls the model again.

The result message looks like this:

```json
{ "role": "tool", "tool_call_id": "call_abc123", "content": "pub fn login(user: &str) -> Result<Session> { ... }" }
```

The ID is how the model matches each result to its request. On the next call, the model sees its own request followed by the result, and continues from there. So every round trip adds at least two entries to the history: the call and the result. Here is how the same pieces are named across providers:

| | OpenAI Chat Completions | OpenAI Responses | Anthropic Messages | Google Gemini |
|---|---|---|---|---|
| The call | `tool_calls` on the assistant message | a `function_call` item | a `tool_use` content block | a `functionCall` part |
| How you know | `finish_reason: "tool_calls"` | `function_call` items in `output` | `stop_reason: "tool_use"` | `functionCall` parts in the reply |
| Arguments | a JSON string | a JSON string | a JSON object, `input` | a JSON object, `args` |
| The result | a `role: "tool"` message | a `function_call_output` item | a `tool_result` block in a user message | a `functionResponse` part |
| Linked by | `tool_call_id` | `call_id` | `tool_use_id` | `id`, plus the name |
| Error flag | none, put it in the text | none | `is_error: true` | none |

**How a text predictor produces JSON.** The same way it produces everything else: as tokens. Open models are trained to write tool calls inside special markers, and the inference server watches for them. A Hermes-style model such as Qwen writes `<tool_call>{"name": "get_weather", "arguments": {"city": "Paris"}}</tool_call>`, which is exactly the format its catalog prompt asked for in section 4. OpenAI's harmony format instead addresses the message to the tool:

```text
<|start|>assistant<|channel|>commentary to=functions.get_current_weather <|constrain|>json<|message|>{"location":"San Francisco"}<|call|>
```

The `<|call|>` token is a stop token, just like an end-of-turn token, and that's why a tool call ends the model's turn. The server then parses the text back into the JSON your program receives. The vLLM inference server's Hermes parser literally finds the tags with a regular expression and runs a JSON parser on what's inside. If parsing fails, it returns the whole output as ordinary text. To prevent broken calls, providers can use constrained decoding, which OpenAI and Anthropic call strict mode. It blocks any token that would break the schema, so the arguments are guaranteed to parse, though not guaranteed to be sensible.

**Three details that matter in practice:**

- **Errors are results.** If a tool fails, the harness sends the error text back as the tool result instead of crashing. Anthropic marks it with `is_error: true` and says Claude will usually retry two or three times with corrections. Good harnesses do the same for calls to tools that don't exist and for arguments that won't parse, because the model can usually fix its own mistake once it sees what went wrong.
- **Calls can come in parallel.** The model can ask for several tools in one turn, such as reading three files at once. The harness may run them at the same time, but it must send back a result for every call. Anthropic wants them all in one user message, and warns that splitting them across several messages "teaches" Claude to stop making parallel calls.
- **Reasoning has to be passed back.** Reasoning models think before they call a tool, and that thinking must go back to the API on the next request. Anthropic requires thinking blocks to be passed back "complete and unmodified." OpenAI asks for its reasoning items to be passed back too, in encrypted form if you don't store data with them.

Tool results don't have to be text, either. Anthropic's API and OpenAI's Responses API accept images and documents in tool results, which is how a browser tool can hand the model a screenshot.

### 6. MCP: tools from somewhere else

Every tool so far was written into the harness itself. That doesn't scale. If ten AI apps each want to talk to GitHub, Slack, and a database, someone has to write thirty integrations. The Model Context Protocol, which Anthropic introduced in November 2024, changes the math: a tool provider writes one MCP server, and any app that speaks MCP can use it. It caught on quickly. In December 2025 Anthropic donated MCP to the new Agentic AI Foundation, a fund under the Linux Foundation co-founded with Block and OpenAI, and by then there were about 10,000 active servers. Claude, ChatGPT, Codex, Cursor, VS Code, and Gemini CLI all support it.

**The three roles.** MCP names the parts like this:

- **Host:** the AI app the user works in, such as Claude Code or Cursor. In this tutorial's terms, the harness.
- **Client:** a connection object inside the host. There is exactly one client per server.
- **Server:** a separate program that offers tools, and a few other things, over the protocol.

Messages use JSON-RPC 2.0, a simple request-and-response format, over one of two transports. With stdio, the host starts the server as a child process, and they exchange one JSON message per line over its standard input and output. With Streamable HTTP, the server is a web service and each message is an HTTP POST, which is how remote servers and hosted connectors work.

**The flow for tools.** For tools, the protocol boils down to two requests. First the host asks what's available:

```json
{ "jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {} }
```

The server answers with definitions that look a lot like the ones in section 4:

```json
{ "jsonrpc": "2.0", "id": 1, "result": { "tools": [{
  "name": "get_weather",
  "description": "Get current weather information for a location",
  "inputSchema": { "type": "object", "properties": { "location": { "type": "string" } }, "required": ["location"] }
}]}}
```

When the model calls that tool, the host forwards the call:

```json
{ "jsonrpc": "2.0", "id": 2, "method": "tools/call", "params": { "name": "get_weather", "arguments": { "location": "New York" } } }
```

And the server replies with content blocks and an error flag:

```json
{ "jsonrpc": "2.0", "id": 2, "result": { "content": [{ "type": "text", "text": "72°F, partly cloudy" }], "isError": false } }
```

These examples are trimmed. The current spec also expects a small `_meta` block on each request and a few extra fields in each result.

**The glue the host provides.** The model never sees MCP. It sees ordinary tools in its catalog, and the host does the translation in both directions:

1. At startup, the host calls `tools/list` on every configured server.
2. It turns each MCP tool into a normal catalog entry and renames it to avoid collisions. Claude Code, for example, uses `mcp__<server>__<tool>`, so GitHub's repository search becomes `mcp__github__search_repositories`.
3. When the model calls one, the host looks up which server owns it and sends `tools/call` with the original name.
4. It turns the server's content blocks into a tool result tagged with the model's call ID. The JSON-RPC `id` and the model's tool call ID are separate numbers, and the host keeps track of which goes with which.
5. If a server sends `notifications/tools/list_changed`, the host fetches the list again.

**More than tools.** Tools are the main attraction, but MCP defines a few other primitives, each controlled by a different party:

| Primitive | Offered by | Controlled by | What it's for |
|---|---|---|---|
| Tools | Server | The model | Actions the model can call |
| Resources | Server | The application | Data the app can attach to the context, such as files or records |
| Prompts | Server | The user | Reusable prompt templates, often shown as slash commands |
| Elicitation | Client | The user | Lets a server ask the user a question in the middle of a task |

Two older client features, sampling (letting a server borrow the host's model) and roots (telling a server which folders it may use), were deprecated in the current spec.

**A recent change to watch for.** Older material shows a handshake at the start of every connection: the client sends `initialize`, the server replies with its capabilities, and the client confirms. The current spec, version 2026-07-28, removed that handshake to make MCP stateless. The changelog says it plainly: "Make MCP stateless: remove the `initialize`/`notifications/initialized` handshake." Now every request declares its own protocol version and the client's capabilities in its `_meta` field, and servers must answer a `server/discover` request, which clients may send first if they want to check versions up front. Many servers and docs still use the older handshake, and the spec allows a server to support both.

**The cost problem.** MCP makes it easy to plug in a lot of tools, and that is its main practical problem. Every server's tools land in the catalog, so the context costs from section 4 pile up. That's why hosts now load MCP tools on demand. Anthropic went further in its "Code execution with MCP" post (November 2025), which presents MCP tools to the model as code it can call from a sandbox. In one workflow, that cut usage "from 150,000 tokens to 2,000 tokens."

**The trust problem.** The deeper issue is trust. An MCP server's tool descriptions and results go straight into the model's context, and the model can't reliably tell data from instructions. The known attacks:

- **Tool poisoning:** hidden instructions in a tool's description, invisible in the app's interface but read by the model.
- **Rug pulls:** a server changes a tool's description after the user approved it.
- **Injection through results:** in one demonstrated attack, a malicious public GitHub issue got an agent to leak private repository data into a public pull request, without any tool being compromised.
- **The lethal trifecta:** Simon Willison's warning that an agent with access to private data, exposure to untrusted content, and a way to send data out can be tricked into handing that data to an attacker.

### 7. The tool loop

One tool call rarely finishes anything. Fixing a bug means running the tests, reading the code, editing it, and running the tests again, and each step depends on what the last one showed. So the harness wraps everything in a loop. Simon Willison's definition of an agent is the shortest one going: "An LLM agent runs tools in a loop to achieve a goal." Here is that loop, assembled from Anthropic's and OpenAI's documentation:

```python
messages = [user_message]
for turn in range(MAX_TURNS):
    response = model(system_prompt, tools, messages)    # streamed to the screen
    messages.append(response.assistant_message)

    if not response.tool_calls:          # a plain answer: the task is done
        return response

    results = []
    for call in response.tool_calls:     # independent calls may run in parallel
        if allowed(call):                # rules, approvals, hooks
            results.append(run_with_timeout(call))
        else:
            results.append(denied(call)) # the model sees the refusal and adapts
    messages.append(tool_results(results))   # every result, matched by ID

raise TooManyTurns
```

That really is the engine of every agent. The rest of a harness exists to feed this loop well and keep it safe.

**One bug fix, step by step.** Here is what the loop looks like on a small bug fix. Each row is one call to the model:

| Model call | What the model asks for | What gets appended | Messages in the request |
|---|---|---|---|
| 1 | Run the test suite | The call, then the failing test output | 1 |
| 2 | Read `auth.rs` | The call, then the file's contents | 3 |
| 3 | Edit one function | The call, then "edit applied" | 5 |
| 4 | Run the tests again | The call, then "all tests passed" | 7 |
| 5 | Nothing. It replies "Fixed the off-by-one in token expiry." | The final answer | 9 |

That's five model calls for one user request, and each call resends everything before it, including the full test output and the whole file. The system prompt and tool catalog go along on every one of those calls, which is why prompt caching matters so much for agents. OpenAI's write-up of the Codex loop says a single user turn can involve "hundreds of tool calls," and Codex is built so that each request is an exact prefix of the next one, keeping the cache warm the whole way.

**How the loop ends.** The stop reason on each response tells the harness what to do next:

| Stop signal | What it means | What the harness does |
|---|---|---|
| `tool_use` (Anthropic), `tool_calls` (OpenAI) | The model wants tools | Run them, append the results, call again |
| `end_turn`, `stop` | The model is finished | Show the answer and wait for the user |
| `max_tokens`, `length` | The reply hit the output limit | Continue, or retry with a higher limit. A tool call cut off halfway is useless. |
| `pause_turn` (Anthropic) | A server-side tool loop hit its own cap | Send the request again so it can continue |
| `refusal`, `content_filter` | The model declined | Show it. Resending the same request usually gets the same answer. |
| `model_context_window_exceeded` (Anthropic) | The input filled the context window | Compact and retry (section 8) |

**Brakes.** A loop that could run forever needs brakes, and harnesses use several:

- **Turn limits.** A cap on model calls per request. Defaults vary widely (see the table below).
- **Budgets and timeouts.** Token or dollar budgets, per-tool timeouts (Claude Code's shell tool defaults to two minutes), and deadlines for the whole run.
- **Loop detection.** Gemini CLI flags five identical tool calls in a row, and OpenCode asks the user after three.
- **Quitting checks.** Harnesses can push back when the model stops too early. Claude Code's Stop hook, for example, can tell it to keep going.

| SDK or harness | Default turn limit | What happens at the limit |
|---|---|---|
| OpenAI Agents SDK | 10 turns | Raises `MaxTurnsExceeded` |
| Vercel AI SDK `ToolLoopAgent` | 20 steps | Stops and logs a warning |
| Claude Agent SDK | None, unless you set `max_turns` | Returns an `error_max_turns` result |
| LangGraph | 10,007 steps (raised from 25 in 2026) | Raises `GraphRecursionError` |
| Gemini CLI | Unlimited by default | Ends the session if you set a limit |

**Permissions, sandboxes, and hooks.** The loop is also where the harness enforces safety, because every action passes through it. The model asks, and the harness decides. Two designs show the range:

- **Claude Code uses permission modes.** They run from `default` (ask before edits and commands), through `acceptEdits`, `plan` (read-only), and `auto` (a classifier reviews each action), to `bypassPermissions`. Rules are checked deny first, then ask, then allow, and the docs stress that they are "enforced by Claude Code, not by the model."
- **Codex CLI separates what's possible from when to ask.** The sandbox mode (`read-only`, `workspace-write`, or `danger-full-access`) limits what commands can touch at the operating system level. The approval policy decides when to stop and ask the user.

Hooks let users extend this with scripts that run at fixed points in the loop. A Claude Code `PreToolUse` hook can block a call before it runs, and Codex and Cursor have similar hooks.

**Streaming, interrupting, and nesting.** Two more features make the loop feel interactive. Streaming sends text to the screen as it's generated, so you can watch the agent work. Interruption lets you stop it: in Claude Code, Esc stops the current response or tool, and anything you type while it works is delivered as soon as the current tool calls finish. Stopping has limits, though. As OpenAI's SDK docs put it, "Cancellation cannot undo tool effects that have already happened."

Finally, loops can nest. A sub-agent is a tool whose implementation is another agent loop with its own fresh message list. Claude Code's Agent tool (formerly called Task) hands a job to a sub-agent, which might spend tens of thousands of tokens exploring and then return a summary of a thousand or two. The parent's list only grows by that summary. That makes sub-agents one of the best ways to keep context small, which is the subject of the next section.

### 8. Compaction

The loop has a side effect: the message list only ever grows. Every file read, search result, and test log stays in it. Two limits eventually catch up with it. The hard limit is the context window. Send more than it holds and the API simply rejects the request. The soft limit arrives much earlier, because models get worse at using information as their input gets longer. A 2023 study, "Lost in the Middle," found that models use information at the start and end of a long input much better than information in the middle. Chroma's 2025 "Context Rot" study of 18 models found that performance "consistently degrades with increasing input length," even when the task itself gets no harder. Anthropic describes this as an "attention budget" that every token spends a little of.

**The options.** Harnesses have several ways to keep the list under control, and each one trades something away:

| Strategy | How it works | What it keeps | What it loses |
|---|---|---|---|
| Truncation | Drop the oldest messages | The system prompt and recent turns, word for word | Everything older, possibly including the original task |
| Clearing tool results | Replace old tool outputs with a placeholder | Every message, and a record that each call happened | The raw output, which can usually be fetched again |
| Summarization | Have the model summarize the history, then swap the summary in | Goals, decisions, file names, next steps, recent turns | Exact wording and numbers, images, and the prompt cache |
| Sub-agents | Do the messy work in a separate loop | A clean parent context plus a short report | The sub-agent's evidence, and total token use goes up |
| Notes files | The agent writes its progress to disk | Anything written down, across resets and sessions | Whatever it didn't write down |
| Just-in-time retrieval | Load data only when it's needed | A small context up front | Time, and anything the agent never thinks to look up |

**How summarization works.** When people say "compaction," they usually mean the summarization approach. It goes like this:

1. Before each request, the harness estimates how full the context is.
2. Past a threshold, it sends the history to the model with a special instruction: write a summary someone else could pick the work up from.
3. It replaces the old messages with that summary, leaving the system prompt untouched and usually keeping the most recent messages too.
4. It restores anything essential, such as rules files and, in Claude Code's case, the most recently edited files.
5. The loop continues on the shorter list.

Here is the entire compaction prompt from OpenAI's open-source Codex CLI:

> You are performing a CONTEXT CHECKPOINT COMPACTION. Create a handoff summary for another LLM that will resume the task.
>
> Include:
> - Current progress and key decisions made
> - Important context, constraints, or user preferences
> - What remains to be done (clear next steps)
> - Any critical data, examples, or references needed to continue
>
> Be concise, structured, and focused on helping the next LLM seamlessly continue the work.

The phrase "another LLM" is the right way to think about it. After compaction, the model is effectively a new worker reading a handoff note from the previous one.

**How real harnesses do it.** As of November 2026:

| Harness | When it compacts | How | What you control |
|---|---|---|---|
| Claude Code | At the model's limit by default (about 967K tokens on 1M-token models) | A model-written summary, then it re-reads up to five recently edited files and puts CLAUDE.md back | `/compact` with optional instructions, `/clear`, `/autocompact` |
| Codex CLI | At 90% of the window | The provider's encrypted compaction when available, otherwise the prompt above plus up to 20,000 tokens of recent user messages | `/compact`, `/clear`, the token threshold, a custom prompt |
| Gemini CLI | At 50% of the window | An XML `<state_snapshot>` summary, checked by a second pass, keeping the newest 30% of the history | `/compress`, `/clear`, a threshold setting |
| OpenCode | At about 85% full | A structured summary plus about 15,000 recent tokens | Automatic on or off, how much to keep, which model writes the summary |

**Compaction inside the APIs.** Compaction is also moving into the providers' APIs. Anthropic offers context editing, which clears old tool results automatically (by default once input passes 100,000 tokens, keeping the last three), and server-side compaction that writes the summary for you and drops everything before it. OpenAI's Responses API has a compaction option and a standalone `/responses/compact` endpoint, and both return an encrypted item that "is opaque and not intended to be human-interpretable." OpenAI even trains models for it. It described GPT-5.1-Codex-Max as its "first model natively trained to operate across multiple context windows through a process called compaction."

**What compaction costs.** It's a necessary tool, not a free one:

- **It breaks the cache.** The rewritten history no longer matches any cached prefix. In one OpenCode report, pruning old tool outputs dropped cached tokens from about 219,000 to zero and raised the cost of a single request from about $0.24 to $1.83.
- **Details get lost.** A summary keeps the gist, not the exact error message or line number, so the agent often has to read files again.
- **Summaries drift.** Repeated compaction produces a summary of a summary, and errors build up. Gemini CLI now updates its previous snapshot instead of starting over, specifically to prevent "information drift."
- **The timing is bad.** As one Anthropic article notes, "due to context rot, the model is at its least intelligent point when compacting."

That's why long-running harnesses don't rely on compaction alone. Anthropic's harness for multi-session coding has the agent keep a progress file, a JSON list of features, and a clean git history, so a fresh session can rebuild its understanding from disk. In Anthropic's words, "compaction isn't sufficient."

### 9. Putting it together: that's a harness

Step back and look at what we've built. The model takes a stream of tokens and predicts more tokens. Everything else came from the program around it: the message list, the system prompt, the tool catalog, parsing and running tool calls, MCP connections, the loop and its brakes, and compaction. That program is the harness. LangChain's summary is the neatest: "Agent = Model + Harness," and "If you're not the model, you're the harness." Anthropic describes it as "the loop, tools, context management, and guardrails that turn raw intelligence into a working agent."

**What a full harness takes care of.** Production harnesses take on a long list of jobs. Here is the inventory, with a real example of each:

| Responsibility | What it means | A real example |
|---|---|---|
| Conversation state | Keeping the message list and deciding what goes in it | Every harness |
| Prompt assembly | Building the system prompt, rules files, and environment context | Codex CLI: base instructions plus AGENTS.md |
| Tool registry | Merging built-in, MCP, and plugin tools into one catalog | Claude Code's `mcp__server__tool` naming |
| Tool execution | Parsing, validating, running, and returning tool calls | Every harness |
| Permissions | Deciding what needs the user's approval | Claude Code's permission modes |
| Sandboxing | Limiting what commands can touch at the OS level | Codex CLI's sandbox modes |
| Hooks | Running user scripts at fixed points in the loop | Claude Code's 33 hook events |
| Sub-agents | Running nested loops with their own context | Claude Code's Agent tool |
| Context management | Clearing, compacting, and summarizing | The harnesses in section 8 |
| Persistence | Saving, resuming, and forking sessions | Codex's App Server threads |
| Checkpoints | Rewinding file changes | Claude Code's `/rewind` |
| Observability | Recording traces of model calls and tool calls | OpenAI Agents SDK tracing |
| Cost accounting | Tracking tokens and spending | Claude Code's `/usage` |

**The harness matters as much as the model.** It's tempting to think the model does the real work and the harness is plumbing. Benchmarks say otherwise. On Terminal-Bench 2.0, the same Claude Opus 4.6 model scored 58.0% inside Claude Code and 81.8% inside a different harness, ForgeCode. Part of that gap came from broken tasks that a later version of the benchmark fixed (Claude Code's score rose to 70.1%), but the harness effect is real. LangChain raised one fixed model from 52.8% to 66.5% on the same benchmark by changing only its harness. The labs also train models inside their harnesses. OpenAI says its Codex models were trained to excel at the `apply_patch` editing format, and Anthropic calls the schemas of its built-in tools "trained-in." A model and its harness are increasingly designed as a pair.

**The core is small.** And yet the heart of it is tiny. Thorsten Ball's "How to Build an Agent" builds a working code-editing agent in under 400 lines of Go, most of it boilerplate: "It's an LLM, a loop, and enough tokens." mini-SWE-agent solves more than 74% of SWE-bench Verified with a single bash tool and an agent file under 200 lines of Python. Production harnesses are big because of everything around the loop: permissions, sandboxes, compaction, the interface, and a long tail of edge cases. If you want to read real ones, these are open source:

| Harness | Maker | License | Language | Known for |
|---|---|---|---|---|
| Codex CLI | OpenAI | Apache-2.0 | Rust | One harness behind every Codex product, with a sandbox on by default |
| Gemini CLI | Google | Apache-2.0 | TypeScript | GEMINI.md files, loop detection, checkpointing |
| OpenCode | Anomaly | MIT | TypeScript | Separate build and plan agents |
| Aider | Aider-AI | Apache-2.0 | Python | Repository map, automatic git commits, architect and editor models |
| Goose | Block, now the Linux Foundation | Apache-2.0 | Rust | A general-purpose agent built around MCP extensions |
| Cline | Cline | Apache-2.0 | TypeScript | Plan and act modes, approval for every action, checkpoints |

That's the whole picture. A harness is a loop that keeps a list of messages, shows the model what it can do, does what the model asks, and keeps the list small enough to stay useful. The model brings the judgment. The harness brings the memory, the hands, and the brakes.

## Where Runvil fits

Runvil is a harness with an unusual shape, and the pieces above map onto it closely. Most harnesses hard-code the loop. In Runvil, the prompt author writes it: a prompt is a Markdown file whose Lua code blocks decide when to call the model, which tools to offer, and how to compact. The other unusual choice is a split down the middle of the harness. The Engine steps through the prompt and does no I/O of its own. Whenever it needs a model reply, a tool result, a timer, or a file, it emits a request, called an effect, and waits. The Harness performs each effect and records it. Workshop, the desktop app, is the Host that embeds the Harness. The Gateway is a separate process that holds the API keys and serves the OpenAI-compatible format from section 1.

The built-in Workshop chat agent shows most of this tutorial in a few lines of Lua. This is its main loop, from `crates/workshop/agents/agents/chat.md`, with the compaction function left out:

```lua
tools.offer("user-input")
tools.offer("std")
tools.offer("web")
tools.offer(tools.packs.extras())

while true do
    local text = input.ask()
    messages:user(text)
    pcall(function() return models.loop(compact) end)
end
```

`tools.offer` builds the tool catalog, `messages:user` appends to the message list, and `models.loop` runs the tool loop. `compact` is the prompt's own compaction function. It summarizes everything before the last two user turns with a separate `models.infer` call and swaps that summary in as a single message.

| Concept | In Runvil | Where to look |
|---|---|---|
| Message list | Each section's `messages` list, edited from Lua with `messages:user`, `messages:replace`, and similar calls, then converted to OpenAI-shaped messages right before each model call | `crates/runvil-internal/lua/src/messages.rs` |
| System prompt | Only what the prompt adds with `messages:system(...)`. Runvil adds none of its own. | The same file |
| Tool catalog | Packs, offered with `tools.offer`. Each Pack renders its tools at the section's first loop round, and the catalog is frozen after that. | `crates/runvil-plugin/src/pack.rs`, `crates/runvil-internal/lua/src/tools.rs` |
| Tool calls | The Gateway client parses the streamed reply, the Engine rejects unknown tool names, and the Harness runs the owning Pack's `call` | `crates/harness-gateway-client/src/wire/`, `crates/harness-internal/runner/src/performers.rs` |
| MCP | A single `mcp` Pack for remote Streamable HTTP servers, exposed through two discovery tools | `crates/plugin-mcp/src/lib.rs` |
| Tool loop | `models.loop`, a Rust state machine, with 24 model rounds per call by default (`max_tool_iterations`) and up to 10 tool calls running at once | `crates/runvil-internal/lua/src/models_loop.rs`, `crates/harness-internal/runner/src/effect_loop.rs` |
| Compaction | A compactor function the loop calls when the next request won't fit. The default, `compactors.fail`, doesn't compact at all. | `crates/runvil-internal/lua/src/compactors.rs` |
| Run recording | Every effect and its answer, appended to Workshop's run log | `crates/workshop/run-log` |

A few differences from the mainstream harnesses are worth knowing before you read the code:

- **No hidden system prompt.** Runvil adds none, and the chat agent's own system message line is currently commented out, so today the chat agent sends no system prompt at all.
- **MCP tools are discovered, not listed.** The `mcp` Pack exposes `get_dynamic_tools` and `call_dynamic_tool` instead of copying every server's tools into the catalog. It's the deferred-loading idea from section 4. Local stdio servers aren't supported yet.
- **Compaction is the prompt's decision.** The loop detects the overflow, estimating about four characters per token or using the provider's last reported usage, but what to keep is up to the compactor the prompt passes in.
- **Model roles are declared but not yet routed.** Prompts can ask for `default`, `strong`, `small`, `fast`, or `creative`, but today the Harness fills only `default`, so in Workshop every role uses the model selected in the dropdown.

## Glossary

| Term | Meaning |
|---|---|
| Token | A chunk of text, about four characters of English on average, that models read and write |
| Context window | The most tokens a model can take in at once |
| Chat template | The rules that flatten a list of messages into one token stream with role markers |
| Prefill and decode | The two phases of generation: reading the prompt in parallel, then writing one token at a time |
| Prompt caching | Reusing the processed prefix of a recent request, at a discount |
| System prompt | The developer's instructions at the start of the conversation |
| Tool catalog | The list of tool definitions sent with every request |
| Tool call | The model's structured request to use a tool, with a name, arguments, and an ID |
| Tool result | A tool's output, sent back to the model with the matching ID |
| MCP | The Model Context Protocol, a standard way to connect AI apps to tool servers |
| Tool loop | Call the model, run its tools, append the results, and repeat until it answers |
| Compaction | Shrinking the history, usually by replacing old messages with a summary |
| Sub-agent | A nested agent loop with its own fresh context, used like a tool |
| Harness | Everything around the model that turns it into an agent |

## Further reading

**Start here**

- [I think "agent" may finally have a widely enough agreed upon definition to be useful jargon now](https://simonwillison.net/2025/Sep/18/agents/) by Simon Willison, the one-line definition of an agent, explained.
- [How to Build an Agent](https://ampcode.com/how-to-build-an-agent) by Thorsten Ball, a working agent in under 400 lines.
- [The Anatomy of an Agent Harness](https://www.langchain.com/blog/the-anatomy-of-an-agent-harness) from LangChain, on where the model ends and the harness begins.

**The conversation and the system prompt**

- [Chat templates](https://huggingface.co/docs/transformers/main/en/chat_templating) from Hugging Face, on how messages become tokens.
- [OpenAI Harmony response format](https://cookbook.openai.com/articles/openai-harmony), every token of a real chat format, tools included.
- [Prompt caching](https://docs.anthropic.com/en/docs/build-with-claude/prompt-caching) from Anthropic, on why prefix order matters.

**Tools and MCP**

- [How tool use works](https://platform.claude.com/docs/en/agents-and-tools/tool-use/how-tool-use-works) from Anthropic, and [Function calling](https://platform.openai.com/docs/guides/function-calling) from OpenAI.
- [Writing effective tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) from Anthropic.
- [MCP architecture overview](https://modelcontextprotocol.io/docs/learn/architecture) and the [2026-07-28 spec changelog](https://modelcontextprotocol.io/specification/2026-07-28/changelog).
- [The lethal trifecta](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) by Simon Willison.

**The loop and compaction**

- [Unrolling the Codex agent loop](https://openai.com/index/unrolling-the-codex-agent-loop/) from OpenAI.
- [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) from Anthropic.
- [Context Rot](https://research.trychroma.com/context-rot) from Chroma.
- [Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) from Anthropic.

*2026-11-06 07:27 - Claude Opus 5.5*
