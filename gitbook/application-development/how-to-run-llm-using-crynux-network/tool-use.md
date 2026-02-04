# Tool Use (Function Calling)

Crynux Bridge supports the standard OpenAI Tool Use (Function Calling) API. This allows you to describe functions to the model, and have the model intelligently choose to output a JSON object containing arguments to call one or many of those functions.

{% hint style="warning" %}
It is important to note that **not all open-source models support tool calling**. You must choose a model that has been specifically trained or fine-tuned for this capability.

Generally, models with "Instruct" in their name (Instruction Fine-Tuned models) are more likely to support tool use. For example, if you are using the Qwen model family, the base model `Qwen/Qwen2.5-7B` might not support tool calls effectively, whereas the instruction-tuned version `Qwen/Qwen2.5-7B-Instruct` is designed to handle such tasks.

Always check the model card or documentation of the specific model you intend to use to confirm its support for function calling or tool use.
{% endhint %}

The following examples demonstrate how to use the tool calling feature with the **OpenAI SDK**, the dedicated **langchain-crynux** library, and the standard **langchain-openai** library.

{% tabs %}
{% tab title="OpenAI SDK" %}
When using the official `openai` Python SDK, you define tools as dictionaries and pass them to the `tools` parameter. The `vram_limit` is passed via `extra_body`.

```python
from openai import OpenAI
import json

client = OpenAI(
    base_url="https://bridge.crynux.io/v1/llm",
    api_key="your-api-key",
)

# 1. Define the tool
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_current_weather",
            "description": "Get the current weather in a given location",
            "parameters": {
                "type": "object",
                "properties": {
                    "location": {
                        "type": "string",
                        "description": "The city and state, e.g. San Francisco, CA",
                    },
                    "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]},
                },
                "required": ["location"],
            },
        },
    }
]

# 2. Call the model
messages = [{"role": "user", "content": "What's the weather like in Boston today?"}]
completion = client.chat.completions.create(
    model="Qwen/Qwen2.5-7B-Instruct",
    messages=messages,
    tools=tools,
    tool_choice="auto",
    extra_body={
        "vram_limit": 24
    }
)

response_message = completion.choices[0].message
tool_calls = response_message.tool_calls

if tool_calls:
    print("Tool calls detected:")
    for tool_call in tool_calls:
        print(f"Function: {tool_call.function.name}")
        print(f"Arguments: {tool_call.function.arguments}")
```
{% endtab %}

{% tab title="LangChain-Crynux" %}
The `langchain-crynux` library provides a drop-in replacement for `ChatOpenAI` optimized for Crynux. It handles `vram_limit` as a first-class parameter.

```python
from langchain_crynux import ChatCrynux
from langchain_core.tools import tool

# 1. Define the tool using the @tool decorator
@tool
def get_current_weather(location: str, unit: str = "celsius"):
    """Get the current weather in a given location"""
    # Simulate a weather API response
    return {
        "location": location,
        "temperature": "22",
        "unit": unit,
        "condition": "Sunny"
    }

# 2. Initialize the ChatCrynux model
llm = ChatCrynux(
    base_url="https://bridge.crynux.io/v1/llm",
    model="Qwen/Qwen2.5-7B-Instruct",
    vram_limit=24,           # Specify VRAM requirement directly
    api_key="your-api-key"
)

# 3. Bind the tool to the model
llm_with_tools = llm.bind_tools([get_current_weather])

# 4. Invoke the model
query = "What's the weather like in Boston today?"
response = llm_with_tools.invoke(query)

print("Tool Calls:", response.tool_calls)
```
{% endtab %}

{% tab title="LangChain-OpenAI" %}
If you prefer using the standard `langchain-openai` library, you can pass the Crynux-specific `vram_limit` parameter inside the `model_kwargs` dictionary.

```python
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

# 1. Define the tool
@tool
def get_current_weather(location: str, unit: str = "celsius"):
    """Get the current weather in a given location"""
    return {
        "location": location,
        "temperature": "22",
        "unit": unit,
        "condition": "Sunny"
    }

# 2. Initialize ChatOpenAI with Crynux configuration
llm = ChatOpenAI(
    base_url="https://bridge.crynux.io/v1/llm",
    api_key="your-api-key",
    model="Qwen/Qwen2.5-7B-Instruct",
    # Pass Crynux parameters in model_kwargs
    model_kwargs={
        "vram_limit": 24
    }
)

# 3. Bind and invoke
llm_with_tools = llm.bind_tools([get_current_weather])
response = llm_with_tools.invoke("What's the weather like in Boston today?")

print("Tool Calls:", response.tool_calls)
```
{% endtab %}
{% endtabs %}
