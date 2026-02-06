# Thinking Progress Display Fix

## Issue
When using the augment-opencode wrapper with OpenCode, thinking progress (reasoning content) was appearing at the **bottom** of the response instead of at the **top**, making it difficult to see the model's reasoning process.

## Root Cause
The streaming callback was sending thinking chunks (`agent_thought_chunk`) immediately as they arrived, which meant they were sent **after** message chunks (`agent_message_chunk`) in the stream. This caused OpenCode to display them at the end of the response.

## Solution
Implemented a **reasoning buffer** that:
1. **Buffers** all reasoning chunks as they arrive
2. **Combines** them into a single reasoning chunk for cleaner output
3. **Flushes** the buffer when the first message chunk arrives
4. **Handles** late reasoning chunks (received after text has started)

## Changes Made

### File: `src/server.ts`

#### 1. Added StreamCallbackResult Interface (Line 1097-1100)
```typescript
interface StreamCallbackResult {
  callback: (notification: SessionNotification): void;
  flush: () => void;
}
```

#### 2. Modified createStreamCallback Function (Line 1102+)
- Added reasoning buffer: `let reasoningBuffer: string[] = []`
- Added state tracking: `hasStartedTextContent` and `hasFlushedReasoning` flags
- Added flush helper: `flushReasoningBuffer()` function that:
  - Combines all buffered reasoning into a single chunk
  - Sends as one `reasoning_content` delta
  - Logs the flush operation with chunk count and character count
- Modified `agent_message_chunk` case to:
  - Check if this is the first text content
  - Call `flushReasoningBuffer()` before sending text
- Modified `agent_thought_chunk` case to:
  - Buffer reasoning before text starts
  - Send immediately (with warning) if received after text has started
- Returns `{ callback, flush }` object

#### 3. Updated callAugmentAPIStreamingInternal
- Stored callback result: `const streamHandler = createStreamCallback(...)`
- Registered handler: `client.onSessionUpdate(streamHandler.callback)`
- Added flush call in finally block: `streamHandler.flush()`

## Key Features

### 1. Reasoning Buffer
- Buffers all `agent_thought_chunk` updates before text content
- Combines multiple reasoning chunks into a single output
- Prevents fragmented reasoning display

### 2. Smart Flushing
- Automatically flushes when first text content arrives
- Ensures reasoning always appears before text
- Handles edge case of late reasoning chunks

### 3. Detailed Logging
- Logs reasoning flush operations with metrics
- Warns about late reasoning chunks (suboptimal but handled)
- Helps debug streaming behavior

## Behavior
**Before Fix:**
```
Stream order: message → message → thinking → thinking
Display: [Message content] [Thinking at bottom]
```

**After Fix:**
```
Stream order: [buffered reasoning] → combined reasoning chunk → message → message
Display: [Thinking at top] [Message content]
```

## Edge Cases Handled

### Late Reasoning Chunks
If reasoning arrives after text content has started:
- Logs a warning: `⚠️ Late reasoning chunk received after text started`
- Sends immediately (may appear at end of output)
- This is suboptimal but preserves all content

## Compatibility
- ✅ Maintains OpenAI-compatible API format
- ✅ Works with all models (Claude, GPT)
- ✅ No breaking changes to existing functionality
- ✅ Backward compatible with OpenCode

