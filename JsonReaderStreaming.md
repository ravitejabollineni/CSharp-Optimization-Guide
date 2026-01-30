# Utf8JsonStreamReaderHelper

*A deep-dive explanation for .NET developers – rewritten for clarity, correctness, and architectural understanding.*

---

## 1. What Problem Does This Solve?

When you read JSON from a **stream** (file, network, HTTP response), you **cannot assume**:

- The full JSON object arrives in one read
- Objects align with buffer boundaries
- You can load everything into memory

`Utf8JsonStreamReaderHelper` solves this by:

- Reading JSON **incrementally** from a `Stream`
- Storing bytes in **pooled memory segments**
- Feeding a `Utf8JsonReader` using `ReadOnlySequence<byte>`
- Extracting **exactly one JSON value** at a time for `JsonSerializer`

This is *low-level*, allocation-conscious, and designed for **high-throughput streaming scenarios**.

---

## 2. High-Level Architecture

```
Input Stream
    ↓
[ Memory Segment A ] → [ Memory Segment B ] → [ Memory Segment C ]
        ↓                        ↓                    ↓
         └────────── ReadOnlySequence<byte> ──────────┘
                                ↓
                     Utf8JsonReader (stateful)
                                ↓
                 JsonSerializer.Deserialize<T>()
```

Key idea: **JSON tokens may span multiple buffers**, but `ReadOnlySequence<byte>` makes them look contiguous.

---

## 3. Why `ref struct`?

```csharp
public ref struct Utf8JsonStreamReaderHelper
```

This is deliberate:

- Prevents heap allocation
- Ensures stack-only lifetime
- Avoids accidental async usage

This enforces **correct usage with streaming readers** and avoids GC pressure.

---

## 4. Core Fields – What Each One Really Means

### Stream & Buffer Control

```csharp
private readonly Stream _stream;
private readonly int _bufferSize;
```

- `_stream`: source of UTF-8 JSON bytes
- `_bufferSize`: *minimum* buffer size (actual rented buffers may be larger)

⚠️ Never use `_bufferSize` as a hard limit.

---

### Segment Tracking

```csharp
private SequenceSegment? _firstSegment;
private int _firstSegmentStartIndex;
private SequenceSegment? _lastSegment;
private int _lastSegmentEndIndex;
```

These define the **active window** of data:

```
Segment A        Segment B        Segment C
|----used----|--active data--|--free space--|
      ↑                          ↑
 firstSegmentStartIndex     lastSegmentEndIndex
```

Only bytes between these indices are visible to the reader.

---

### Parsing State

```csharp
private Utf8JsonReader _jsonReader;
private bool _keepBuffers;
private bool _isFinalBlock;
```

- `_jsonReader`: stateful JSON tokenizer
- `_keepBuffers`: prevents disposal during multi-pass scans
- `_isFinalBlock`: signals end-of-stream to the reader

---

## 5. Reading Tokens – `Read()`

```csharp
public bool Read()
```

This method guarantees:

- If JSON token is incomplete → read more data
- If stream is exhausted → stop safely

### Control Flow

```
Try reader.Read()
   ↓ success → return true
   ↓ failure
        ↓
   IsFinalBlock?
        ↓ yes → return false
        ↓ no
   MoveNext() → read more bytes
```

This loop is **critical** for handling tokens split across buffers.

---

## 6. Buffer Expansion – `MoveNext()`

This method:

1. Advances past consumed bytes
2. Disposes fully-used segments
3. Rents a new pooled buffer
4. Reads more bytes from the stream
5. Rebuilds `ReadOnlySequence<byte>`
6. Recreates `Utf8JsonReader` *with preserved state*

### Why recreate the reader?

Because `Utf8JsonReader`:

- Is a struct
- Does **not** grow its input
- Requires a new instance when data expands

But state continuity is preserved via:

```csharp
_jsonReader.CurrentState
```

---

## 7. Object Boundary Detection – `DeserialisePre()`

### Why this exists

`JsonSerializer.Deserialize<T>` can read **only one JSON value**.

But the stream may contain:

```
{...}{...}{...}
```

So we must *extract exactly one value*.

---

### How It Works

1. Capture token start position
2. Enable `_keepBuffers`
3. Track **nesting depth**

```text
StartObject / StartArray → depth++
EndObject   / EndArray   → depth--
```

4. Stop when depth returns to zero

This guarantees a **complete JSON value**, even if nested.

---

## 8. Actual Deserialization – `Deserialise<T>()`

```csharp
var seq = sequence.Slice(tokenStart, reader.Position);
var newReader = new Utf8JsonReader(seq, true, default);
JsonSerializer.Deserialize<T>(ref newReader);
```

Important details:

- A **new reader** is created
- It sees only the object slice
- Parsing happens exactly once

No copying. No parsing twice.

---

## 9. Memory Cleanup – `DeserialisePost()`

After deserialization:

- Consumed segments are disposed
- Start index is advanced
- Reader is rebuilt if necessary

This ensures:

- Minimal memory retention
- Safe reuse of pooled buffers

---

## 10. SequenceSegment – Why It Exists

```csharp
private sealed class SequenceSegment : ReadOnlySequenceSegment<byte>
```

Purpose:

- Wrap pooled memory
- Participate in `ReadOnlySequence<byte>`
- Track `Previous` for iterative disposal

### Disposal Strategy

```text
Segment C
  ↓
Segment B
  ↓
Segment A
```

Disposed **iteratively**, not recursively → avoids stack overflow.

---

## 11. Memory Layout (Visual)

```
[ Segment A ] → [ Segment B ] → [ Segment C ]
      ↑               ↑              ↑
   consumed        parsing        unread
```

Only the active window is exposed to the reader.

---

## 12. Design Guarantees

✅ Zero-copy parsing

✅ Pooled memory (low GC pressure)

✅ Handles arbitrarily large JSON

✅ Correct across buffer boundaries

✅ Safe disposal

---

## 13. When Should You Use This?

Use this pattern when:

- Parsing large JSON streams
- Reading concatenated JSON objects
- Building high-performance ingestion pipelines

Avoid it when:

- JSON fits comfortably in memory
- Simplicity is more important than throughput

---

## 14. Final Summary

`Utf8JsonStreamReaderHelper` is a **carefully engineered streaming JSON parser** that:

- Bridges `Stream` → `Utf8JsonReader` → `JsonSerializer`
- Manages memory manually and safely
- Extracts one JSON value at a time

This is **expert-level infrastructure code**, and your implementation is architecturally sound.

---

*End of document*

