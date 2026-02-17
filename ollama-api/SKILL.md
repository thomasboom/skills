---
name: ollama-api
description: "Use this skill whenever the user wants to write code, generate examples, debug requests, or answer questions involving the Ollama REST API or Ollama Cloud API. Triggers include: any mention of 'Ollama API', 'ollama.com/api', 'localhost:11434', '/api/generate', '/api/chat', '/api/embed', pulling/pushing/creating Ollama models programmatically, or integrating Ollama into Python/JavaScript/curl workflows. Also use when the user asks about Ollama's OpenAI-compatible endpoints, streaming responses, structured output, tool calling, or embeddings. Do NOT use for general LLM theory, non-Ollama APIs, or Ollama CLI commands (use the Ollama docs instead)."
---

# Ollama API Skill

A skill for writing correct, idiomatic code that calls the Ollama REST API — both the local server (`http://localhost:11434`) and the Ollama Cloud (`https://ollama.com/api`).

---

## Quick Reference

| Task | Endpoint | Method |
|------|----------|--------|
| Generate text (single prompt) | `/api/generate` | POST |
| Chat (multi-turn conversation) | `/api/chat` | POST |
| Embeddings | `/api/embed` | POST |
| List available models | `/api/tags` | GET |
| List running models | `/api/ps` | GET |
| Show model details | `/api/show` | POST |
| Pull a model | `/api/pull` | POST |
| Push a model | `/api/push` | POST |
| Create a model | `/api/create` | POST |
| Copy a model | `/api/copy` | POST |
| Delete a model | `/api/delete` | DELETE |
| Get server version | `/api/version` | GET |

---

## Base URLs

```
# Local (default after installation)
http://localhost:11434/api

# Ollama Cloud
https://ollama.com/api
```

The `OLLAMA_HOST` environment variable overrides the local base URL.

---

## Authentication

Local Ollama servers run unauthenticated by default.

For **Ollama Cloud** (`ollama.com`), authentication uses ED25519 key signing. The official Python and JavaScript clients handle this automatically. When calling the cloud API directly, pass your API key:

```bash
curl https://ollama.com/api/chat \
  -H "Authorization: Bearer $OLLAMA_API_KEY" \
  -d '{ "model": "llama3.2", "messages": [{"role": "user", "content": "Hello"}] }'
```

---

## Model Name Format

Models follow `model:tag` format. Examples:
- `llama3.2` (defaults to `llama3.2:latest`)
- `llama3.2:3b`
- `llama3:70b`
- `orca-mini:3b-q4_1`
- `example/custom-model:v1`

---

## Streaming

Most inference endpoints stream responses by default (`"stream": true`). Each chunk is a newline-delimited JSON object (NDJSON). The final object has `"done": true` and includes usage statistics.

Disable streaming for a single response object:
```json
{ "stream": false }
```

All durations in responses are in **nanoseconds**.

---

## Endpoints

### POST /api/generate — Generate a Response

Generate a completion for a single prompt.

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | ✅ | Model name |
| `prompt` | string | — | The prompt text |
| `suffix` | string | — | Text after the model response (for fill-in-middle) |
| `images` | array | — | Base64-encoded images (multimodal models only) |
| `format` | string/object | — | `"json"` or a JSON Schema for structured output |
| `options` | object | — | Model parameters (see Model Options below) |
| `system` | string | — | Overrides the system message from Modelfile |
| `template` | string | — | Overrides the prompt template from Modelfile |
| `stream` | bool | — | Default `true`. Set `false` for single response |
| `raw` | bool | — | Skip template formatting; send raw prompt |
| `keep_alive` | string | — | How long model stays in VRAM (default `"5m"`) |
| `think` | bool | — | Enable extended thinking for reasoning models |

**Streaming example (curl):**
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Why is the sky blue?"
}'
```

**Non-streaming example:**
```bash
curl http://localhost:11434/api/generate -d '{
  "model": "llama3.2",
  "prompt": "Why is the sky blue?",
  "stream": false
}'
```

**Response fields (final/non-streaming):**
```json
{
  "model": "llama3.2",
  "created_at": "2024-08-04T19:22:45.499127Z",
  "response": "The sky is blue because...",
  "done": true,
  "done_reason": "stop",
  "context": [1, 2, 3],
  "total_duration": 4935648287,
  "load_duration": 534986918,
  "prompt_eval_count": 26,
  "prompt_eval_duration": 107345000,
  "eval_count": 237,
  "eval_duration": 4289432000
}
```

---

### POST /api/chat — Generate a Chat Message

Multi-turn conversational endpoint. The caller maintains conversation history.

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | ✅ | Model name |
| `messages` | array | ✅ | Conversation history (see Message format below) |
| `tools` | array | — | Tools for function calling (if model supports it) |
| `format` | string/object | — | `"json"` or a JSON Schema |
| `options` | object | — | Model parameters |
| `stream` | bool | — | Default `true` |
| `keep_alive` | string | — | VRAM retention duration (default `"5m"`) |
| `think` | bool | — | Enable thinking for reasoning models |

**Message format:**
```json
{
  "role": "user | assistant | system | tool",
  "content": "message text",
  "images": ["<base64>"],
  "tool_calls": []
}
```

**Important:** The chat endpoint is **stateless**. You must send the full conversation history with every request.

**Non-streaming example:**
```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "stream": false,
  "messages": [
    { "role": "system", "content": "You are a helpful assistant." },
    { "role": "user", "content": "Why is the sky blue?" }
  ]
}'
```

**Response:**
```json
{
  "model": "llama3.2",
  "created_at": "2023-12-12T14:13:43.416799Z",
  "message": {
    "role": "assistant",
    "content": "The sky is blue due to Rayleigh scattering..."
  },
  "done": true,
  "total_duration": 5191566416,
  "eval_count": 298
}
```

**Multi-turn pattern (Python):**
```python
from ollama import chat

messages = [
    {"role": "system", "content": "You are a helpful assistant."},
    {"role": "user", "content": "What is the capital of France?"},
]

response = chat(model="llama3.2", messages=messages)

# Append assistant reply to maintain history
messages.append({"role": "assistant", "content": response.message.content})

# Continue the conversation
messages.append({"role": "user", "content": "What's the tallest building there?"})
response2 = chat(model="llama3.2", messages=messages)
```

---

### POST /api/embed — Generate Embeddings

Generate vector embeddings for one or more inputs.

**Parameters:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | string | ✅ | Model name |
| `input` | string or array | ✅ | Text(s) to embed |
| `truncate` | bool | — | Truncate inputs to context length (default `true`) |
| `options` | object | — | Model parameters |
| `keep_alive` | string | — | VRAM retention |

**Example:**
```bash
curl http://localhost:11434/api/embed -d '{
  "model": "nomic-embed-text",
  "input": ["Why is the sky blue?", "What is the weather like today?"]
}'
```

**Response:**
```json
{
  "model": "nomic-embed-text",
  "embeddings": [
    [0.010071029, -0.0017594862, 0.05007221, ...],
    [0.0098027075, 0.0091999006, 0.0482856, ...]
  ],
  "total_duration": 274309234,
  "prompt_eval_count": 8
}
```

---

### GET /api/tags — List Models

List all models available locally.

```bash
curl http://localhost:11434/api/tags
```

**Response:**
```json
{
  "models": [
    {
      "name": "llama3.2:latest",
      "model": "llama3.2:latest",
      "modified_at": "2023-11-04T14:56:49.277302595-07:00",
      "size": 7365960935,
      "digest": "9f438cb9cd581...",
      "details": {
        "format": "gguf",
        "family": "llama",
        "parameter_size": "3B",
        "quantization_level": "Q4_0"
      }
    }
  ]
}
```

---

### GET /api/ps — List Running Models

Show which models are currently loaded in VRAM.

```bash
curl http://localhost:11434/api/ps
```

**Response:**
```json
{
  "models": [
    {
      "name": "llama3.2:latest",
      "model": "llama3.2:latest",
      "size": 5137025024,
      "size_vram": 5137025024,
      "digest": "9f438cb9cd581...",
      "expires_at": "2024-06-04T14:38:31.83753-07:00"
    }
  ]
}
```

---

### POST /api/pull — Pull a Model

Download a model from the Ollama library. Cancelled pulls resume automatically.

```bash
curl http://localhost:11434/api/pull -d '{
  "model": "llama3.2"
}'
```

Streaming status objects:
```json
{ "status": "pulling manifest" }
{ "status": "downloading digestname", "digest": "sha256:...", "total": 2142590208, "completed": 241970 }
{ "status": "verifying sha256 digest" }
{ "status": "writing manifest" }
{ "status": "success" }
```

Parameters: `model` (required), `insecure` (allow insecure connections, dev only), `stream`.

---

### POST /api/push — Push a Model

Upload a model to the Ollama library. Requires an ollama.com account with a public key registered.

```bash
curl http://localhost:11434/api/push -d '{
  "model": "mattw/pygmalion:latest"
}'
```

Parameters: `model` (required, must be `<namespace>/<model>:<tag>` format), `insecure`, `stream`.

---

### POST /api/create — Create a Model

Create a model from a Modelfile definition.

```bash
curl http://localhost:11434/api/create -d '{
  "model": "my-custom-model",
  "from": "llama3.2",
  "system": "You are Mario from Super Mario Bros.",
  "parameters": {
    "temperature": 0.7
  }
}'
```

---

### POST /api/copy — Copy a Model

Duplicate a model under a new name.

```bash
curl http://localhost:11434/api/copy -d '{
  "source": "llama3.2",
  "destination": "llama3-backup"
}'
```

---

### DELETE /api/delete — Delete a Model

Remove a model and its data.

```bash
curl -X DELETE http://localhost:11434/api/delete -d '{
  "model": "llama3.2"
}'
```

---

### GET /api/version — Get Server Version

```bash
curl http://localhost:11434/api/version
# {"version":"0.5.1"}
```

---

## Model Options

These go inside the `"options"` field on generate/chat requests:

```json
{
  "options": {
    "temperature": 0.8,
    "top_p": 0.9,
    "top_k": 40,
    "num_predict": 128,
    "num_ctx": 4096,
    "seed": 42,
    "stop": ["\n", "User:"],
    "repeat_penalty": 1.1,
    "presence_penalty": 0.0,
    "frequency_penalty": 0.0
  }
}
```

| Option | Description |
|--------|-------------|
| `temperature` | Creativity (0.0–2.0, default ~0.8). Lower = more deterministic |
| `top_p` | Nucleus sampling threshold |
| `top_k` | Limit vocab to top-K tokens |
| `num_predict` | Max tokens to generate (-1 = infinite) |
| `num_ctx` | Context window size (default varies by model) |
| `seed` | For reproducible outputs |
| `stop` | Stop sequences — generation halts when matched |
| `repeat_penalty` | Penalty for repeating tokens |

---

## Structured Output

### JSON mode
```json
{
  "model": "llama3.2",
  "prompt": "List three planets as JSON",
  "format": "json",
  "stream": false
}
```

### JSON Schema (structured output)
```json
{
  "model": "llama3.2",
  "prompt": "Return a person object",
  "format": {
    "type": "object",
    "properties": {
      "name": { "type": "string" },
      "age": { "type": "integer" }
    },
    "required": ["name", "age"]
  },
  "stream": false
}
```

---

## Tool Calling (Function Calling)

Supported on models that advertise tool support. Provide tools in the chat request:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llama3.2",
  "stream": false,
  "messages": [{"role": "user", "content": "What is the weather in Toronto?"}],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_current_weather",
        "description": "Get the current weather for a city",
        "parameters": {
          "type": "object",
          "properties": {
            "city": { "type": "string", "description": "The city name" }
          },
          "required": ["city"]
        }
      }
    }
  ]
}'
```

The model may respond with a tool call:
```json
{
  "message": {
    "role": "assistant",
    "content": "",
    "tool_calls": [
      {
        "function": {
          "name": "get_current_weather",
          "arguments": { "city": "Toronto" }
        }
      }
    ]
  }
}
```

After executing the tool, append the result as a `tool` role message and call the API again.

---

## Official Libraries

### Python
```bash
pip install ollama
```
```python
from ollama import chat, generate, Client

# Simple generate
response = generate(model='llama3.2', prompt='Why is the sky blue?')
print(response.response)

# Chat
response = chat(model='llama3.2', messages=[
    {'role': 'user', 'content': 'Why is the sky blue?'}
])
print(response.message.content)

# Custom host / cloud
client = Client(host='https://ollama.com', headers={'Authorization': 'Bearer TOKEN'})
response = client.chat(model='llama3.2', messages=[...])
```

### JavaScript / TypeScript
```bash
npm install ollama
```
```javascript
import ollama from 'ollama';

// Generate
const response = await ollama.generate({
  model: 'llama3.2',
  prompt: 'Why is the sky blue?',
  stream: false,
});
console.log(response.response);

// Chat
const chatResponse = await ollama.chat({
  model: 'llama3.2',
  messages: [{ role: 'user', content: 'Why is the sky blue?' }],
});
console.log(chatResponse.message.content);

// Custom host
import { Ollama } from 'ollama';
const client = new Ollama({ host: 'https://ollama.com' });
```

### Streaming in JavaScript
```javascript
const stream = await ollama.chat({
  model: 'llama3.2',
  messages: [{ role: 'user', content: 'Tell me a story.' }],
  stream: true,
});

for await (const chunk of stream) {
  process.stdout.write(chunk.message.content);
}
```

---

## OpenAI Compatibility

Ollama exposes an OpenAI-compatible layer. Drop it into any OpenAI SDK by changing the base URL:

**Base URL:** `http://localhost:11434/v1` (local) or `https://ollama.com/v1` (cloud)

```python
from openai import OpenAI

client = OpenAI(
    base_url='http://localhost:11434/v1',
    api_key='ollama',  # required by SDK but ignored locally
)

response = client.chat.completions.create(
    model='llama3.2',
    messages=[{'role': 'user', 'content': 'Why is the sky blue?'}]
)
print(response.choices[0].message.content)
```

**Compatible endpoints:**
- `POST /v1/chat/completions` → `/api/chat`
- `POST /v1/completions` → `/api/generate`
- `POST /v1/embeddings` → `/api/embed`
- `GET /v1/models` → `/api/tags`

---

## Multimodal (Vision) Models

For models like `llava` or `llama3.2-vision`, pass images as base64 strings:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "llava",
  "stream": false,
  "messages": [
    {
      "role": "user",
      "content": "What is in this image?",
      "images": ["<base64-encoded-image-data>"]
    }
  ]
}'
```

```python
import base64, ollama

with open("image.jpg", "rb") as f:
    img_b64 = base64.b64encode(f.read()).decode()

response = ollama.chat(
    model='llava',
    messages=[{
        'role': 'user',
        'content': 'Describe this image.',
        'images': [img_b64]
    }]
)
```

---

## Extended Thinking (Reasoning Models)

For reasoning-capable models, enable the `think` parameter:

```bash
curl http://localhost:11434/api/chat -d '{
  "model": "deepseek-r1:7b",
  "stream": false,
  "think": true,
  "messages": [{"role": "user", "content": "Solve: 25 * 48"}]
}'
```

The thinking content appears in the `thinking` field of the response message.

---

## keep_alive — VRAM Control

Control how long a model stays loaded after a request:

```json
{ "keep_alive": "10m" }   // 10 minutes
{ "keep_alive": "1h" }    // 1 hour
{ "keep_alive": 0 }       // Unload immediately after response
{ "keep_alive": -1 }      // Keep loaded indefinitely
```

Default is `"5m"`. Accepts duration strings (`"5m"`, `"1h"`) or nanoseconds as a number.

---

## Error Handling

Errors return standard HTTP status codes with a JSON body:

```json
{ "error": "model not found" }
```

| Status | Meaning |
|--------|---------|
| `200` | Success |
| `400` | Bad request (invalid JSON, missing required fields) |
| `404` | Model not found |
| `500` | Internal server error |

Always check for `"error"` in the JSON response, and handle connection errors for streaming (the stream may be interrupted).

---

## Common Patterns

### Check if model is available before calling
```python
import ollama

def ensure_model(model_name):
    models = [m.model for m in ollama.list().models]
    if model_name not in models:
        print(f"Pulling {model_name}...")
        ollama.pull(model_name)
```

### Simple chatbot loop (Python)
```python
from ollama import chat

messages = []
while True:
    user_input = input("You: ")
    messages.append({"role": "user", "content": user_input})
    response = chat(model="llama3.2", messages=messages)
    reply = response.message.content
    messages.append({"role": "assistant", "content": reply})
    print(f"Assistant: {reply}")
```

### Batch embeddings for RAG
```python
import ollama

texts = ["Document one content", "Document two content", "Query text"]
result = ollama.embed(model="nomic-embed-text", input=texts)
embeddings = result.embeddings  # List of float arrays
```

---

## Critical Rules

- **Always send full conversation history** on every `/api/chat` call — the endpoint is stateless.
- **Stream is on by default** — set `"stream": false` if you want a single response object.
- **Model names are case-sensitive** and must match exactly (e.g., `llama3.2`, not `Llama3.2`).
- **All durations in responses are nanoseconds** — divide by `1e9` to get seconds.
- **Use `keep_alive: 0`** to immediately free VRAM after a one-off request.
- **For cloud API calls**, always include the `Authorization: Bearer <token>` header.
- **Images must be base64-encoded** — do not pass file paths.
- **Tool calls are not streamed** — set `"stream": false` when using tools.
