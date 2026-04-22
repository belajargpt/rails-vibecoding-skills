---
name: rubyllm-patterns
description: Use when integrating LLM APIs (Claude, OpenAI, Gemini) in Rails apps via the RubyLLM gem. Applies to AI chatbots, streaming responses, auto-categorization / classification, structured outputs, cost-aware model selection, rate limiting, graceful degradation. Triggers when user mentions AI, LLM, Claude API, OpenAI, Gemini, chatbot, streaming response, auto-categorize, classification, RubyLLM, Anthropic, Net::HTTP AI, or any language model integration in Rails.
---

# RubyLLM Patterns

> Verified against official docs at https://rubyllm.com/ — ruby_llm gem.

## Philosophy

> One interface, all providers. No Net::HTTP plumbing. Cost-aware by default.

RubyLLM gem abstracts LLM APIs behind a single interface: Claude (Anthropic), OpenAI, Gemini. Swap provider = change one parameter.

## Installation

```ruby
# Gemfile
gem "ruby_llm"
```

```bash
bundle install
```

## Configuration

API keys via ENV vars (or Rails credentials, then map to ENV):

```ruby
# config/initializers/ruby_llm.rb
RubyLLM.configure do |config|
  config.anthropic_api_key = ENV.fetch("ANTHROPIC_API_KEY", nil)
  config.openai_api_key = ENV.fetch("OPENAI_API_KEY", nil)
end
```

Only configure providers you use. Set ENV vars via `.env` in dev, Kamal secrets in production.

## Rails Integration (Generators)

If you want full chat UI out of the box:
```bash
bin/rails generate ruby_llm:install      # ActiveRecord Chat + Message models
bin/rails generate ruby_llm:chat_ui      # controllers + Turbo streaming views
```

Access: `http://localhost:3000/chats`.

For custom integration, skip the UI generator and build manually.

## Basic Usage

```ruby
chat = RubyLLM.chat
response = chat.ask("What is Ruby on Rails?")
puts response.content
```

Response is a `RubyLLM::Message` with conversation history automatically maintained.

## Model Selection

```ruby
# On creation
chat = RubyLLM.chat(model: "claude-sonnet-4-6")
chat = RubyLLM.chat(model: "claude-haiku")
chat = RubyLLM.chat(model: "gpt-5-nano")
chat = RubyLLM.chat(model: "gemini-3.1-pro-preview")

# Change model on existing chat
chat.with_model("claude-sonnet-4-6")
```

## System Prompt / Instructions

```ruby
chat = RubyLLM.chat
chat.with_instructions("You are a helpful finance assistant...")
chat.ask("What's my biggest expense category?")

# Replace
chat.with_instructions("New instruction here")

# Append
chat.with_instructions("Answer in one paragraph.", append: true)
```

## Streaming Response (Block Syntax)

```ruby
chat = RubyLLM.chat(model: "claude-sonnet-4-6")

final = chat.ask("Explain Rails in 3 paragraphs") do |chunk|
  print chunk.content  # fires per token/chunk
end

puts final.content  # full accumulated response
```

Chunk attributes: `content`, `role`, `model_id`, `tool_calls`, `input_tokens`, `output_tokens`.

### Rails Turbo Stream Integration
```ruby
class ChatStreamJob < ApplicationJob
  def perform(chat_id, user_message, stream_target_id)
    chat = Chat.find(chat_id)
    full_response = ""

    chat.ask(user_message) do |chunk|
      full_response << (chunk.content || "")
      Turbo::StreamsChannel.broadcast_replace_to(
        "chat_#{chat.id}",
        target: stream_target_id,
        partial: "messages/streaming_message",
        locals: { content: full_response }
      )
    end
  end
end
```

## Structured Output (Schema)

```ruby
# Class-based
class TransactionCategorySchema < RubyLLM::Schema
  string :category, description: "Category: food, transport, entertainment"
  number :confidence, description: "Confidence 0.0-1.0"
end

chat = RubyLLM.chat(model: "claude-haiku")
response = chat.with_schema(TransactionCategorySchema)
  .ask("Classify: 'Kopi Starbucks 50K'")

# response.content matches schema structure
```

Or manual Hash schema:
```ruby
schema = {
  type: "object",
  properties: {
    category: { type: "string", enum: %w[food transport entertainment] },
    confidence: { type: "number" }
  },
  required: %w[category confidence]
}
chat.with_schema(schema).ask("Classify: ...")
```

Remove schema: `chat.with_schema(nil)`.

## Cost-Aware Model Selection

Pick cheapest model per task:

| Task | Model | Relative cost |
|---|---|---|
| Classification | `claude-haiku` | $ |
| Chat with reasoning | `claude-sonnet-4-6` | $$ |
| Complex reasoning | `claude-opus-4-7` | $$$ |

Token tracking on response:
```ruby
response = chat.ask("...")
puts "Input: #{response.input_tokens}, Output: #{response.output_tokens}"
```

Use for monthly cost math: `total_tokens × model_price_per_token × users`.

## Conversation History

Auto-maintained by Chat instance:
```ruby
chat.ask("Initial question")
chat.ask("Follow-up")  # has context of prior exchange
chat.messages.each { |m| puts "[#{m.role.to_s.upcase}] #{m.content}" }
```

Manual message:
```ruby
chat.add_message(role: :system, content: raw_block)
```

## Event Handlers

```ruby
chat.on_new_message { print "Assistant > " }
chat.on_end_message { |msg| puts "Complete!" }
```

## Graceful Degradation

External API = can fail. Never block core functionality:
```ruby
def auto_categorize!
  return if category.present?

  category_result = RubyLLM.chat(model: "claude-haiku")
    .ask("Category for: #{description}")
  update(category: category_result.content.strip)
rescue RubyLLM::Error => e
  Rails.logger.warn "AI categorize failed: #{e.message}"
  # Fallback: leave blank, user inputs manually
end
```

## Rate Limiting (Cost Control)

Rails 8 built-in:
```ruby
class ChatController < ApplicationController
  rate_limit to: 50, within: 1.day, by: -> { current_user.id }
end
```

## Anti-Patterns

- ❌ `Net::HTTP.post` directly — use RubyLLM
- ❌ Hardcoding API key — use ENV or Rails credentials
- ❌ Always using Opus — use Haiku for simple tasks
- ❌ No rate limit — users drain budget
- ❌ No fallback on API failure — broken UX

## References

- `references/ai-llm.md` — command pattern, cost tracking, tool patterns (37signals style)
- Official docs: https://rubyllm.com/
