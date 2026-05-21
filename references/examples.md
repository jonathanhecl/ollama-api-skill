# Code Examples

Language-agnostic snippets for common Ollama operations.

## cURL

### Version
```bash
curl http://localhost:11434/api/version
```

### List Models
```bash
curl http://localhost:11434/api/tags
```

### Show Model Metadata
```bash
curl http://localhost:11434/api/show -d '{"model":"gemma4:e2b"}'
```

### Chat (Non-Streaming)
```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4:e2b",
  "messages": [{"role": "user", "content": "Say ok"}],
  "stream": false
}'
```

### Chat (Streaming)
```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4:e2b",
  "messages": [{"role": "user", "content": "Say ok"}],
  "stream": true
}'
```

### Vision
```bash
curl http://localhost:11434/api/chat -d '{
  "model": "gemma4:e2b",
  "messages": [{
    "role": "user",
    "content": "Describe this image.",
    "images": ["'$(base64 -w 0 image.jpg)'"]
  }]
}'
```

### Embeddings
```bash
curl http://localhost:11434/api/embed -d '{
  "model": "nomic-embed-text:latest",
  "input": "The quick brown fox jumps over the lazy dog."
}'
```

## Go

### Minimal HTTP Client
```go
package main

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"time"
)

type ChatRequest struct {
	Model    string    `json:"model"`
	Messages []Message `json:"messages"`
	Stream   *bool     `json:"stream,omitempty"`
}

type Message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

type ChatResponse struct {
	Model   string  `json:"model"`
	Message Message `json:"message"`
	Done    bool    `json:"done"`
}

func chat(model, prompt string) (*ChatResponse, error) {
	stream := false
	reqBody, _ := json.Marshal(ChatRequest{
		Model:    model,
		Messages: []Message{{Role: "user", Content: prompt}},
		Stream:   &stream,
	})
	resp, err := http.Post("http://localhost:11434/api/chat", "application/json", bytes.NewReader(reqBody))
	if err != nil {
		return nil, err
	}
	defer resp.Body.Close()
	body, _ := io.ReadAll(resp.Body)
	var out ChatResponse
	if err := json.Unmarshal(body, &out); err != nil {
		return nil, err
	}
	return &out, nil
}

func main() {
	out, err := chat("gemma4:e2b", "Why is the sky blue?")
	if err != nil {
		panic(err)
	}
	fmt.Println(out.Message.Content)
}
```

### Streaming Handler
```go
func streamChat(ctx context.Context, model, prompt string, onChunk func(string)) error {
	stream := true
	payload, _ := json.Marshal(ChatRequest{
		Model:    model,
		Messages: []Message{{Role: "user", Content: prompt}},
		Stream:   &stream,
	})
	req, _ := http.NewRequestWithContext(ctx, "POST", "http://localhost:11434/api/chat", bytes.NewReader(payload))
	req.Header.Set("Content-Type", "application/json")
	resp, err := http.DefaultClient.Do(req)
	if err != nil {
		return err
	}
	defer resp.Body.Close()
	decoder := json.NewDecoder(resp.Body)
	for {
		var chunk ChatResponse
		if err := decoder.Decode(&chunk); err != nil {
			if err == io.EOF {
				return nil
			}
			return err
		}
		onChunk(chunk.Message.Content)
		if chunk.Done {
			return nil
		}
	}
}
```

## Python

### Native HTTP
```python
import requests

resp = requests.post("http://localhost:11434/api/chat", json={
    "model": "gemma4:e2b",
    "messages": [{"role": "user", "content": "Why is the sky blue?"}],
    "stream": False
})
print(resp.json()["message"]["content"])
```

### Streaming
```python
import requests

resp = requests.post("http://localhost:11434/api/chat", json={
    "model": "gemma4:e2b",
    "messages": [{"role": "user", "content": "Count to 10"}],
    "stream": True
}, stream=True)

for line in resp.iter_lines():
    if line:
        chunk = json.loads(line)
        print(chunk["message"]["content"], end="")
        if chunk.get("done"):
            break
```

### OpenAI SDK
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

completion = client.chat.completions.create(
    model="gemma4:e2b",
    messages=[{"role": "user", "content": "Say this is a test"}],
)
print(completion.choices[0].message.content)
```

### Structured Outputs (Pydantic)
```python
from pydantic import BaseModel
from openai import OpenAI

class FriendInfo(BaseModel):
    name: str
    age: int
    is_available: bool

class FriendList(BaseModel):
    friends: list[FriendInfo]

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
completion = client.beta.chat.completions.parse(
    temperature=0,
    model="gemma4:e2b",
    messages=[{"role": "user", "content": "Return a list of friends in JSON format"}],
    response_format=FriendList,
)
print(completion.choices[0].message.parsed)
```

### Embeddings (OpenAI SDK)
```python
from openai import OpenAI

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")
embeddings = client.embeddings.create(
    model="nomic-embed-text:latest",
    input=["why is the sky blue?", "why is the grass green?"],
)
```

### Vision with OpenAI SDK
```python
from openai import OpenAI
import base64

client = OpenAI(base_url="http://localhost:11434/v1", api_key="ollama")

with open("image.jpg", "rb") as f:
    b64 = base64.b64encode(f.read()).decode()

response = client.chat.completions.create(
    model="gemma4:e2b",
    messages=[{
        "role": "user",
        "content": [
            {"type": "text", "text": "What's in this image?"},
            {"type": "image_url", "image_url": {"url": f"data:image/jpeg;base64,{b64}"}},
        ]
    }],
)
print(response.choices[0].message.content)
```

## JavaScript / TypeScript

### Native Fetch
```javascript
const resp = await fetch("http://localhost:11434/api/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "gemma4:e2b",
    messages: [{ role: "user", content: "Why is the sky blue?" }],
    stream: false,
  }),
});
const data = await resp.json();
console.log(data.message.content);
```

### OpenAI SDK
```javascript
import OpenAI from "openai";

const openai = new OpenAI({
  baseURL: "http://localhost:11434/v1",
  apiKey: "ollama",
});

const completion = await openai.chat.completions.create({
  model: "gemma4:e2b",
  messages: [{ role: "user", content: "Say this is a test" }],
});
console.log(completion.choices[0].message.content);
```

### SSE Streaming
```javascript
const resp = await fetch("http://localhost:11434/api/chat", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({
    model: "gemma4:e2b",
    messages: [{ role: "user", content: "Count to 10" }],
    stream: true,
  }),
});

const reader = resp.body.getReader();
const decoder = new TextDecoder();
let buffer = "";

while (true) {
  const { done, value } = await reader.read();
  if (done) break;
  buffer += decoder.decode(value, { stream: true });
  const lines = buffer.split("\n");
  buffer = lines.pop();
  for (const line of lines) {
    if (!line.trim()) continue;
    const chunk = JSON.parse(line);
    process.stdout.write(chunk.message.content);
    if (chunk.done) return;
  }
}
```

## Official SDKs

These examples use the official `ollama` Python and JavaScript libraries.

### Python Official SDK

#### Chat
```python
import ollama

response = ollama.chat(
    model='gemma4:e2b',
    messages=[{'role': 'user', 'content': 'Why is the sky blue?'}],
)
print(response.message.content)
```

#### Streaming
```python
import ollama

for chunk in ollama.chat(
    model='gemma4:e2b',
    messages=[{'role': 'user', 'content': 'Count to 10'}],
    stream=True,
):
    print(chunk.message.content, end='', flush=True)
```

#### Vision (file path)
```python
import ollama

response = ollama.chat(
    model='llama3.2-vision',
    messages=[{
        'role': 'user',
        'content': 'What is in this image?',
        'images': ['image.jpg'],
    }],
)
print(response.message.content)
```

#### Structured Outputs (Pydantic)
```python
import ollama
from pydantic import BaseModel

class Country(BaseModel):
    name: str
    capital: str
    languages: list[str]

response = ollama.chat(
    model='gemma4:e2b',
    messages=[{'role': 'user', 'content': 'Tell me about Canada.'}],
    format=Country.model_json_schema(),
)
country = Country.model_validate_json(response.message.content)
print(country)
```

#### Tool Calling
```python
import ollama

def add_two_numbers(a: int, b: int) -> int:
    """Add two numbers."""
    return a + b

messages = [{'role': 'user', 'content': 'What is 2 plus 3?'}]
response = ollama.chat(
    model='qwen3',
    messages=messages,
    tools=[add_two_numbers],
)
if response.message.tool_calls:
    for tool in response.message.tool_calls:
        print(tool.function.name, tool.function.arguments)
```

#### Thinking
```python
import ollama

response = ollama.chat(
    model='deepseek-r1',
    messages=[{'role': 'user', 'content': 'What is 10 + 23?'}],
    think=True,
)
print('Thinking:\n' + response.message.thinking)
print('Response:\n' + response.message.content)
```

#### Embeddings
```python
import ollama

response = ollama.embed(
    model='nomic-embed-text',
    input='The quick brown fox jumps over the lazy dog.',
)
print(response.embeddings)
```

### JavaScript Official SDK

#### Chat
```javascript
import ollama from 'ollama';

const response = await ollama.chat({
  model: 'gemma4:e2b',
  messages: [{ role: 'user', content: 'Why is the sky blue?' }],
});
console.log(response.message.content);
```

#### Streaming
```javascript
import ollama from 'ollama';

for await (const chunk of await ollama.chat({
  model: 'gemma4:e2b',
  messages: [{ role: 'user', content: 'Count to 10' }],
  stream: true,
})) {
  process.stdout.write(chunk.message.content);
}
```

#### Structured Outputs (Zod)
```javascript
import ollama from 'ollama';
import { z } from 'zod';
import { zodToJsonSchema } from 'zod-to-json-schema';

const Country = z.object({
  name: z.string(),
  capital: z.string(),
  languages: z.array(z.string()),
});

const response = await ollama.chat({
  model: 'gemma4:e2b',
  messages: [{ role: 'user', content: 'Tell me about Canada.' }],
  format: zodToJsonSchema(Country),
});
const country = Country.parse(JSON.parse(response.message.content));
console.log(country);
```

#### Tool Calling
```javascript
import ollama from 'ollama';

const addTool = {
  type: 'function',
  function: {
    name: 'addTwoNumbers',
    description: 'Add two numbers together',
    parameters: {
      type: 'object',
      required: ['a', 'b'],
      properties: {
        a: { type: 'number', description: 'The first number' },
        b: { type: 'number', description: 'The second number' },
      },
    },
  },
};

for await (const chunk of await ollama.chat({
  model: 'qwen3',
  messages: [{ role: 'user', content: 'What is 2 plus 3?' }],
  tools: [addTool],
  stream: true,
})) {
  if (chunk.message.tool_calls) {
    console.log('Tool call:', chunk.message.tool_calls);
  } else {
    process.stdout.write(chunk.message.content);
  }
}
```

#### Thinking
```javascript
import ollama from 'ollama';

const response = await ollama.chat({
  model: 'deepseek-r1',
  messages: [{ role: 'user', content: 'What is 10 + 23?' }],
  think: true,
});
console.log('Thinking:\n' + response.message.thinking);
console.log('Response:\n' + response.message.content);
```
