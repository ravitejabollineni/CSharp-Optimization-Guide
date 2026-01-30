# Utf8JsonStreamReaderHelper - Comprehensive Guide

## 📋 Overview

`Utf8JsonStreamReaderHelper` is a high-performance, memory-efficient helper class for deserializing UTF-8 JSON objects from streams. It's designed to handle large JSON streams that may not fit entirely in memory by using a **segmented buffer approach** with **memory pooling**.

### Key Benefits
- ✅ **Memory Efficient**: Uses pooled memory segments instead of loading entire stream
- ✅ **Streaming Support**: Handles partial reads and incomplete JSON objects
- ✅ **Zero-Copy**: Minimizes memory allocations using `ReadOnlySequence<byte>`
- ✅ **Progressive Parsing**: Reads only what's needed, releases what's processed

---

## 📝 Complete Source Code

```csharp
using System;
using System.Buffers;
using System.IO;
using System.Text.Json;

[ExcludeFromCodeCoverage]
public ref struct Utf8JsonStreamReaderHelper
{
    private readonly Stream _stream;
    // note: buffers will often be bigger than this - do not ever use this number for calculations.
    private readonly int _bufferSize;

    private SequenceSegment? _firstSegment;
    private int _firstSegmentStartIndex;
    private SequenceSegment? _lastSegment;
    private int _lastSegmentEndIndex;

    private Utf8JsonReader _jsonReader;
    private bool _keepBuffers;
    private bool _isFinalBlock;

    public Utf8JsonStreamReaderHelper(Stream stream, int bufferSize)
    {
        _stream = stream;
        _bufferSize = bufferSize;

        _firstSegment = null;
        _firstSegmentStartIndex = 0;
        _lastSegment = null;
        _lastSegmentEndIndex = -1;

        _jsonReader = default;
        _keepBuffers = false;
        _isFinalBlock = false;
    }

    public bool Read()
    {
        // read could be unsuccessful due to insufficient buffer size, retrying in loop with additional buffer segments
        while (!_jsonReader.Read())
        {
            if (_isFinalBlock)
            {
                return false;
            }

            MoveNext();
        }
        return true;
    }

    private void MoveNext()
    {
        _firstSegmentStartIndex += (int)_jsonReader.BytesConsumed;

        // release previous segments if possible
        while (_firstSegmentStartIndex > 0 && _firstSegment?.Memory.Length <= _firstSegmentStartIndex)
        {
            var currFirstSegment = _firstSegment;
            _firstSegmentStartIndex -= _firstSegment.Memory.Length;
            _firstSegment = (SequenceSegment?)_firstSegment.Next;
            if (!_keepBuffers)
            {
                currFirstSegment.Dispose();
            }
        }

        // create new segment
        var newSegment = new SequenceSegment(_bufferSize, _lastSegment);
        _lastSegment?.SetNext(newSegment);
        _lastSegment = newSegment;

        if (_firstSegment == null)
        {
            _firstSegment = newSegment;
            _firstSegmentStartIndex = 0;
        }

        // read data from stream
        _lastSegmentEndIndex = 0;
        int bytesRead;
        do
        {
            bytesRead = _stream.Read(newSegment.Buffer.Memory.Span.Slice(_lastSegmentEndIndex));
            _lastSegmentEndIndex += bytesRead;
        } while (bytesRead > 0 && _lastSegmentEndIndex < newSegment.Buffer.Memory.Length);

        _isFinalBlock = _lastSegmentEndIndex < newSegment.Buffer.Memory.Length;
        var data = new ReadOnlySequence<byte>(_firstSegment, _firstSegmentStartIndex, _lastSegment, _lastSegmentEndIndex);
        _jsonReader = new Utf8JsonReader(data, _isFinalBlock, _jsonReader.CurrentState);
    }

    private void DeserialisePost()
    {
        // release memory if possible
        var firstSegment = _firstSegment;
        var firstSegmentStartIndex = _firstSegmentStartIndex + (int)_jsonReader.BytesConsumed;

        while (firstSegment?.Memory.Length < firstSegmentStartIndex)
        {
            firstSegmentStartIndex -= firstSegment.Memory.Length;
            firstSegment.Dispose();
            firstSegment = (SequenceSegment?)firstSegment.Next;
        }

        if (firstSegment != _firstSegment)
        {
            _firstSegment = firstSegment;
            _firstSegmentStartIndex = firstSegmentStartIndex;
            var data = new ReadOnlySequence<byte>(_firstSegment!, _firstSegmentStartIndex, _lastSegment!,
                _lastSegmentEndIndex);
            _jsonReader =
                new Utf8JsonReader(data, _isFinalBlock, _jsonReader.CurrentState);
        }
    }

    private long DeserialisePre(out SequenceSegment? firstSegment, out int firstSegmentStartIndex)
    {
        // JsonSerializer.Deserialize can read only a single object. We have to extract
        // object to be deserialized into separate Utf8JsonReader. This incurs one additional
        // pass through data (but data is only passed, not parsed).
        var tokenStartIndex = _jsonReader.TokenStartIndex;
        firstSegment = _firstSegment;
        firstSegmentStartIndex = _firstSegmentStartIndex;

        // loop through data until end of object is found
        _keepBuffers = true;
        int depth = 0;

        if (TokenType == JsonTokenType.StartObject || TokenType == JsonTokenType.StartArray)
            depth++;

        while (depth > 0 && Read())
        {
            if (TokenType == JsonTokenType.StartObject || TokenType == JsonTokenType.StartArray)
                depth++;
            else if (TokenType == JsonTokenType.EndObject || TokenType == JsonTokenType.EndArray)
                depth--;
        }

        _keepBuffers = false;
        return tokenStartIndex;
    }

    public T? Deserialise<T>(JsonSerializerOptions? options = null)
    {
        var tokenStartIndex = DeserialisePre(out var firstSegment, out var firstSegmentStartIndex);

        var seq = new ReadOnlySequence<byte>(firstSegment!, firstSegmentStartIndex, _lastSegment!,
            _lastSegmentEndIndex).Slice(tokenStartIndex, _jsonReader.Position);
        var newJsonReader = new Utf8JsonReader(seq, true, default);

        // deserialize value
        var result = JsonSerializer.Deserialize<T>(ref newJsonReader, options);

        DeserialisePost();
        return result;
    }

    public void Dispose()
    {
        _lastSegment?.Dispose();
        _stream?.Dispose();
        _firstSegment?.Dispose();
    }

    public JsonTokenType TokenType => _jsonReader.TokenType;
    public string? GetString() => _jsonReader.GetString();

    private sealed class SequenceSegment : ReadOnlySequenceSegment<byte>, IDisposable
    {
        internal IMemoryOwner<byte> Buffer { get; }
        internal SequenceSegment? Previous { get; set; }
        private bool _disposed;

        public SequenceSegment(int size, SequenceSegment? previous)
        {
            Buffer = MemoryPool<byte>.Shared.Rent(size);
            Previous = previous;

            Memory = Buffer.Memory;
            RunningIndex = previous?.RunningIndex + previous?.Memory.Length ?? 0;
        }

        public void SetNext(SequenceSegment next) => Next = next;

        public void Dispose()
        {
            // Dispose of this and all previous segments iteratively, not recursively.
            for (var segment = this; segment != null; segment = segment.Previous)
            {
                if (segment._disposed)
                    continue;

                segment._disposed = true;
                segment.Buffer.Dispose();
            }
        }
    }
}
```

---

## 🔬 Detailed Code Walkthrough

### Class Declaration & Fields

```csharp
[ExcludeFromCodeCoverage]
public ref struct Utf8JsonStreamReaderHelper
```

**`ref struct`** - Stack-allocated only, cannot be boxed or stored in fields. This provides:
- ✅ Better performance (no heap allocation)
- ✅ Guaranteed cleanup (stack unwinding)
- ❌ Cannot use in async methods
- ❌ Cannot be a class field

---

#### **Stream Management**
```csharp
private readonly Stream _stream;
private readonly int _bufferSize;
```
- `_stream`: Source of JSON data (file, network, memory stream)
- `_bufferSize`: Suggested size for segments (actual may be larger due to pooling)

---

#### **Segment Tracking**
```csharp
private SequenceSegment? _firstSegment;
private int _firstSegmentStartIndex;
private SequenceSegment? _lastSegment;
private int _lastSegmentEndIndex;
```

**Visual representation:**
```
  _firstSegment                              _lastSegment
       ↓                                          ↓
  [═══════════════════][═════════════][═══════════════]
       ↑                                          ↑
  _firstSegmentStartIndex              _lastSegmentEndIndex
  (offset into first)                  (offset into last)
```

- `_firstSegment`: Start of active data window
- `_firstSegmentStartIndex`: Byte offset into first segment where unread data begins
- `_lastSegment`: End of active data window
- `_lastSegmentEndIndex`: Byte offset into last segment where valid data ends

---

#### **Parser State**
```csharp
private Utf8JsonReader _jsonReader;
private bool _keepBuffers;
private bool _isFinalBlock;
```

- `_jsonReader`: The actual JSON parser that reads tokens
- `_keepBuffers`: When `true`, prevents segment disposal (needed during depth scanning)
- `_isFinalBlock`: Signals end-of-stream (set when read returns less than buffer size)

---

### Constructor

```csharp
public Utf8JsonStreamReaderHelper(Stream stream, int bufferSize)
{
    _stream = stream;
    _bufferSize = bufferSize;

    _firstSegment = null;
    _firstSegmentStartIndex = 0;
    _lastSegment = null;
    _lastSegmentEndIndex = -1;

    _jsonReader = default;
    _keepBuffers = false;
    _isFinalBlock = false;
}
```

**Initialization state:**
```
Stream: [Ready to read]
Segments: null (will be created on first read)
Reader: default (uninitialized)
Flags: keepBuffers=false, isFinalBlock=false
```

---

### Read() Method

```csharp
public bool Read()
{
    // read could be unsuccessful due to insufficient buffer size, 
    // retrying in loop with additional buffer segments
    while (!_jsonReader.Read())
    {
        if (_isFinalBlock)
        {
            return false;
        }

        MoveNext();
    }
    return true;
}
```

**Logic Flow:**
```
┌─────────────────────────┐
│ Try _jsonReader.Read()  │
└─────────────────────────┘
          │
          ├─ Returns true ──→ ✓ Token found, return true
          │
          └─ Returns false ──→ Check if _isFinalBlock
                                │
                                ├─ true ──→ ✗ No more data, return false
                                │
                                └─ false ─→ MoveNext() (fetch more data)
                                            └─→ Loop back and retry
```

**Why the loop?**
- JSON token might span segment boundaries
- Reader needs complete token to return true
- Keep fetching data until token is complete or stream ends

---

### MoveNext() Method - The Heart of the System

This method contains 5 critical phases. Let me break down each one:

#### **Phase 1: Update Consumption Marker**
```csharp
_firstSegmentStartIndex += (int)_jsonReader.BytesConsumed;
```
```
Before:  [{"id":1,"name":"Alice"}...]
          ↑
          startIndex=0

Reader consumed: 24 bytes

After:   [{"id":1,"name":"Alice"}...]
                                 ↑
                                 startIndex=24
```

#### **Phase 2: Release Consumed Segments**
```csharp
while (_firstSegmentStartIndex > 0 && 
       _firstSegment?.Memory.Length <= _firstSegmentStartIndex)
{
    var currFirstSegment = _firstSegment;
    _firstSegmentStartIndex -= _firstSegment.Memory.Length;
    _firstSegment = (SequenceSegment?)_firstSegment.Next;
    
    if (!_keepBuffers)
    {
        currFirstSegment.Dispose();  // Return to pool ♻️
    }
}
```

**Example:**
```
Segment A: 100 bytes, fully consumed (startIndex=150)
Segment B: 100 bytes, partially consumed

Before:
  SegA(100) → SegB(100) → SegC
  ↑
  firstSegment, startIndex=150

Step 1: 150 > 100? Yes, dispose SegA
  startIndex = 150 - 100 = 50
  firstSegment = SegB

After:
  SegB(100) → SegC
  ↑
  firstSegment, startIndex=50
  
SegA → Disposed ♻️
```

#### **Phase 3: Create New Segment**
```csharp
var newSegment = new SequenceSegment(_bufferSize, _lastSegment);
_lastSegment?.SetNext(newSegment);
_lastSegment = newSegment;
```

**Creates doubly-linked structure:**
```
Before:  [SegB] → [SegC]
                   ↑
                   lastSegment

After:   [SegB] → [SegC] → [SegD(new)]
                             ↑
                             lastSegment
                             
SegC ← Previous ─ SegD
SegC ─ Next → SegD
```

#### **Phase 4: Read from Stream**
```csharp
_lastSegmentEndIndex = 0;
int bytesRead;
do
{
    bytesRead = _stream.Read(newSegment.Buffer.Memory.Span.Slice(_lastSegmentEndIndex));
    _lastSegmentEndIndex += bytesRead;
} while (bytesRead > 0 && _lastSegmentEndIndex < newSegment.Buffer.Memory.Length);
```

**Why a loop?**
- Stream.Read() may return less than requested
- Keep reading until buffer full OR stream exhausted

```
Iteration 1: Read 1500 bytes → _lastSegmentEndIndex = 1500
Iteration 2: Read 1500 bytes → _lastSegmentEndIndex = 3000
Iteration 3: Read 1096 bytes → _lastSegmentEndIndex = 4096 (buffer full)
Stop: Buffer full
```

#### **Phase 5: Update Parser**
```csharp
_isFinalBlock = _lastSegmentEndIndex < newSegment.Buffer.Memory.Length;
var data = new ReadOnlySequence<byte>(
    _firstSegment, 
    _firstSegmentStartIndex, 
    _lastSegment, 
    _lastSegmentEndIndex
);
_jsonReader = new Utf8JsonReader(data, _isFinalBlock, _jsonReader.CurrentState);
```
- If we read less than buffer size, we hit end-of-stream
- Create sequence spanning all active segments
- Preserve parser state (important for multi-segment tokens)

---

### DeserialisePre() Method

```csharp
private long DeserialisePre(out SequenceSegment? firstSegment, out int firstSegmentStartIndex)
{
    var tokenStartIndex = _jsonReader.TokenStartIndex;
    firstSegment = _firstSegment;
    firstSegmentStartIndex = _firstSegmentStartIndex;

    _keepBuffers = true;
    int depth = 0;

    if (TokenType == JsonTokenType.StartObject || TokenType == JsonTokenType.StartArray)
        depth++;

    while (depth > 0 && Read())
    {
        if (TokenType == JsonTokenType.StartObject || TokenType == JsonTokenType.StartArray)
            depth++;
        else if (TokenType == JsonTokenType.EndObject || TokenType == JsonTokenType.EndArray)
            depth--;
    }

    _keepBuffers = false;
    return tokenStartIndex;
}
```

**Depth Tracking Visualization:**
```
Token Sequence:         {    "id"  :   1    ,    "obj"  :   {    "x"  :   5    }    }
Depth Changes:          1    1     1   1    1    1      1   2    2     2   2    1    0
                        ↑                                    ↑                        ↑
                    Start depth=1                      Nested object           Complete! depth=0
```

**Example with nested structure:**
```json
{
  "id": 1,
  "items": [
    {"name": "A"},
    {"name": "B"}
  ]
}
```

```
Token           Type            Depth   Action
─────           ────            ─────   ──────
{               StartObject     1       depth++
"id"            PropertyName    1       
1               Number          1       
"items"         PropertyName    1       
[               StartArray      2       depth++
{               StartObject     3       depth++
"name"          PropertyName    3       
"A"             String          3       
}               EndObject       2       depth--
{               StartObject     3       depth++
"name"          PropertyName    3       
"B"             String          3       
}               EndObject       2       depth--
]               EndArray        1       depth--
}               EndObject       0       depth-- → COMPLETE!
```

---

### Deserialise<T>() Method

```csharp
public T? Deserialise<T>(JsonSerializerOptions? options = null)
{
    var tokenStartIndex = DeserialisePre(out var firstSegment, out var firstSegmentStartIndex);

    var seq = new ReadOnlySequence<byte>(
        firstSegment!, 
        firstSegmentStartIndex, 
        _lastSegment!,
        _lastSegmentEndIndex
    ).Slice(tokenStartIndex, _jsonReader.Position);
    
    var newJsonReader = new Utf8JsonReader(seq, isFinalBlock: true, default);
    var result = JsonSerializer.Deserialize<T>(ref newJsonReader, options);

    DeserialisePost();
    return result;
}
```

**Visual Flow:**
```
Full sequence:  [{"id":1,"name":"Alice"}{"id":2,"name":"Bob"}...]
                 ↑                      ↑
            tokenStart              current position

Sliced:         [{"id":1,"name":"Alice"}]
                 └─ Only this object ─┘

New Reader:     Reads ONLY the sliced portion
                ↓
Deserialize:    Person { Id = 1, Name = "Alice" }
                ↓
Cleanup:        Update positions, dispose consumed segments
```

---

### SequenceSegment Inner Class

```csharp
private sealed class SequenceSegment : ReadOnlySequenceSegment<byte>, IDisposable
{
    internal IMemoryOwner<byte> Buffer { get; }
    internal SequenceSegment? Previous { get; set; }
    private bool _disposed;

    public SequenceSegment(int size, SequenceSegment? previous)
    {
        Buffer = MemoryPool<byte>.Shared.Rent(size);
        Previous = previous;
        Memory = Buffer.Memory;
        RunningIndex = previous?.RunningIndex + previous?.Memory.Length ?? 0;
    }

    public void Dispose()
    {
        for (var segment = this; segment != null; segment = segment.Previous)
        {
            if (!segment._disposed)
            {
                segment._disposed = true;
                segment.Buffer.Dispose();
            }
        }
    }
}
```

**Why iterative disposal instead of recursive?**

```csharp
// BAD: Recursive (stack overflow risk with many segments)
Previous?.Dispose();  // ← Could cause stack overflow

// GOOD: Iterative (constant stack space)
for (var segment = this; segment != null; segment = segment.Previous)
{
    segment.Buffer.Dispose();
}
```

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    JSON Stream (Network/File)                   │
│          {"id":1,"name":"Alice"}{"id":2,"name":"Bob"}...        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
                    ┌─────────────────┐
                    │ Read from Stream│
                    └─────────────────┘
                              ↓
        ┌────────────────────────────────────────────┐
        │        Segmented Buffer Chain              │
        │  ┌──────┐   ┌──────┐   ┌──────┐           │
        │  │Seg A │→→→│Seg B │→→→│Seg C │           │
        │  └──────┘   └──────┘   └──────┘           │
        └────────────────────────────────────────────┘
                              ↓
                    ┌─────────────────┐
                    │ Utf8JsonReader  │
                    └─────────────────┘
                              ↓
                    ┌─────────────────┐
                    │ Deserialize<T>  │
                    └─────────────────┘
                              ↓
                    ┌─────────────────┐
                    │  C# Object(s)   │
                    └─────────────────┘
```

---

## 🔍 How It Works: Step-by-Step

### 1️⃣ **Initialization**

```csharp
var helper = new Utf8JsonStreamReaderHelper(stream, bufferSize: 4096);
```

```
Initial State:
┌─────────────────────────────────────────┐
│ _firstSegment:      null                │
│ _lastSegment:       null                │
│ _isFinalBlock:      false               │
│ _keepBuffers:       false               │
└─────────────────────────────────────────┘
```

---

### 2️⃣ **First Read Operation**

When you call `Read()`, the helper:

```
Step 1: Check if reader can advance
┌──────────────────────┐
│ _jsonReader.Read()   │  → Returns false (no data yet)
└──────────────────────┘
              ↓
Step 2: Fetch more data
┌──────────────────────┐
│   MoveNext()         │
└──────────────────────┘
```

---

### 3️⃣ **MoveNext() - The Core Reading Logic**

This is where the magic happens:

```
┌───────────────────────────────────────────────────────────────┐
│ PHASE 1: Release consumed segments                           │
└───────────────────────────────────────────────────────────────┘
  _firstSegmentStartIndex tracks bytes consumed
  
  Before:  [SegA|SegB|SegC]
           ↑
           firstSegment (30 bytes consumed)
  
  After:   [SegB|SegC]
           ↑
           firstSegment (adjusted index)
  
  SegA → Disposed (returned to memory pool)

┌───────────────────────────────────────────────────────────────┐
│ PHASE 2: Create new segment                                  │
└───────────────────────────────────────────────────────────────┘
  
  newSegment = rent 4KB from MemoryPool
  
  Before:  [SegB|SegC]
  
  After:   [SegB|SegC|SegD(new)]
                     ↑
                     lastSegment

┌───────────────────────────────────────────────────────────────┐
│ PHASE 3: Read from stream into new segment                   │
└───────────────────────────────────────────────────────────────┘
  
  do {
    bytesRead = stream.Read(newSegment.Buffer)
    _lastSegmentEndIndex += bytesRead
  } while (bytesRead > 0 && buffer not full)
  
  [SegD: {"id":1,"na... ]  ← Filled with stream data

┌───────────────────────────────────────────────────────────────┐
│ PHASE 4: Create ReadOnlySequence for Utf8JsonReader          │
└───────────────────────────────────────────────────────────────┘
  
  var data = new ReadOnlySequence<byte>(
      _firstSegment,           // Start segment
      _firstSegmentStartIndex, // Start offset
      _lastSegment,            // End segment  
      _lastSegmentEndIndex     // End offset
  )
  
  _jsonReader = new Utf8JsonReader(data, _isFinalBlock, state)
```

---

### 4️⃣ **Segment Chain Visualization**

```
Memory Layout During Processing:
═══════════════════════════════════════════════════════════════

Time T0: Initial read
┌─────────────────────────────────────────┐
│ Segment A (4KB)                         │
│ {"id":1,"name":"Alice"}{"id":2,"n...    │
└─────────────────────────────────────────┘
 ↑                                        ↑
_firstSegment (index: 0)         _lastSegment (index: 4096)


Time T1: After processing first object
                          ┌─────────────────────────────────────────┐
                          │ Segment A (4KB)                         │
                          │ {"id":1,"name":"Alice"}{"id":2,"n...    │
                          └─────────────────────────────────────────┘
                                          ↑                         ↑
                          _firstSegment (index: 24)      _lastSegment
                          
                          ← 24 bytes consumed (first object)


Time T2: Need more data, create Segment B
┌─────────────────────────┐  ┌─────────────────────────┐
│ Segment A (partial)     │→→│ Segment B (new)         │
│ ...me":"Bob"}{"id":3... │  │ ,"name":"Charlie"}...   │
└─────────────────────────┘  └─────────────────────────┘
                         ↑                             ↑
                _firstSegment              _lastSegment


Time T3: Segment A fully consumed and disposed
                              ┌─────────────────────────┐
                              │ Segment B               │
                              │ {"id":3,"name":"Char... │
                              └─────────────────────────┘
                              ↑                         ↑
                     _firstSegment            _lastSegment
                     
Segment A → Returned to MemoryPool ♻️
```

---

## 🎯 Deserialize() - The Complete Process

```csharp
var myObject = helper.Deserialise<MyClass>();
```

### Process Flow

```
┌────────────────────────────────────────────────────────────────┐
│ STEP 1: DeserialisePre()                                      │
│ Purpose: Find the complete JSON object boundaries             │
└────────────────────────────────────────────────────────────────┘

Current Position:  {"id":1,"name":"Alice"}{"id":2...
                   ↑
                   tokenStartIndex

Depth Tracking:
  {           → depth = 1
  "id"        → depth = 1
  :           → depth = 1  
  1           → depth = 1
  ,           → depth = 1
  "name"      → depth = 1
  :           → depth = 1
  "Alice"     → depth = 1
  }           → depth = 0  ✓ Object complete!

_keepBuffers = true  ← Don't dispose segments during scan

┌────────────────────────────────────────────────────────────────┐
│ STEP 2: Slice the sequence                                    │
└────────────────────────────────────────────────────────────────┘

Original:  [{"id":1,"name":"Alice"}{"id":2,"name":"Bob"}...]
           ↑                      ↑
           tokenStartIndex        current position
           
Sliced:    [{"id":1,"name":"Alice"}]
           
var seq = sequence.Slice(tokenStartIndex, _jsonReader.Position)

┌────────────────────────────────────────────────────────────────┐
│ STEP 3: Create dedicated reader                               │
└────────────────────────────────────────────────────────────────┘

var newJsonReader = new Utf8JsonReader(seq, isFinalBlock: true)

This reader ONLY sees: {"id":1,"name":"Alice"}

┌────────────────────────────────────────────────────────────────┐
│ STEP 4: Deserialize                                           │
└────────────────────────────────────────────────────────────────┘

var result = JsonSerializer.Deserialize<MyClass>(ref newJsonReader)

Result: new MyClass { Id = 1, Name = "Alice" }

┌────────────────────────────────────────────────────────────────┐
│ STEP 5: DeserialisePost()                                     │
│ Purpose: Clean up consumed memory                             │
└────────────────────────────────────────────────────────────────┘

_keepBuffers = false  ← Resume normal disposal

Release segments that are now fully consumed
Update reader position to start of next object
```

---

## 🧩 SequenceSegment - The Building Block

```csharp
private sealed class SequenceSegment : ReadOnlySequenceSegment<byte>, IDisposable
{
    internal IMemoryOwner<byte> Buffer { get; }
    internal SequenceSegment? Previous { get; set; }
    // ...
}
```

### Structure

```
┌───────────────────────────────────────────────────────────────┐
│                    SequenceSegment                            │
├───────────────────────────────────────────────────────────────┤
│ Buffer:         IMemoryOwner<byte>  ← Rented from pool       │
│ Memory:         Memory<byte>        ← Actual data view       │
│ RunningIndex:   long                ← Global position        │
│ Next:           SequenceSegment?    ← Forward link           │
│ Previous:       SequenceSegment?    ← Backward link          │
└───────────────────────────────────────────────────────────────┘
```

### Linked Structure

```
Segment A              Segment B              Segment C
┌──────────┐          ┌──────────┐          ┌──────────┐
│ Buffer   │          │ Buffer   │          │ Buffer   │
│ Memory   │   Next→  │ Memory   │   Next→  │ Memory   │
│ RunIdx:0 │  ←Prev   │ RunIdx:  │  ←Prev   │ RunIdx:  │
│          │          │ 4096     │          │ 8192     │
└──────────┘          └──────────┘          └──────────┘
```

### Disposal Chain

```csharp
public void Dispose()
{
    // Iterative disposal (not recursive - prevents stack overflow)
    for (var segment = this; segment != null; segment = segment.Previous)
    {
        if (!segment._disposed)
        {
            segment._disposed = true;
            segment.Buffer.Dispose();  // Return to pool
        }
    }
}
```

**Why iterative?**
- Prevents stack overflow with long chains
- Efficient cleanup of entire segment history

```
Disposal from Segment C:
  
  C → B → A → null
  
  Step 1: Dispose C's buffer → MemoryPool ♻️
  Step 2: Dispose B's buffer → MemoryPool ♻️  
  Step 3: Dispose A's buffer → MemoryPool ♻️
```

---

## 🔄 Complete Example Walkthrough

### Scenario: Stream with 3 JSON objects

```json
{"id":1,"name":"Alice"}{"id":2,"name":"Bob"}{"id":3,"name":"Charlie"}
```

### Execution Flow

```
┌─────────────────────────────────────────────────────────────────┐
│ Iteration 1: Read first object                                 │
└─────────────────────────────────────────────────────────────────┘

1. helper.Read() → Calls MoveNext()
2. Rent Segment A (4KB) from pool
3. stream.Read() → Fills Segment A with: {"id":1,"name":"Alice"}{"id":2...
4. Create Utf8JsonReader over [SegmentA]
5. _jsonReader.Read() → Returns true (found StartObject token)

┌─────────────────────────────────────────────────────────────────┐
│ Iteration 2: Deserialize first object                          │
└─────────────────────────────────────────────────────────────────┘

1. helper.Deserialise<Person>()
2. DeserialisePre() scans: { → depth=1, } → depth=0 ✓
3. Slice sequence: [{"id":1,"name":"Alice"}]
4. JsonSerializer.Deserialize() → Person { Id=1, Name="Alice" }
5. DeserialisePost() → Move _firstSegmentStartIndex to position 24

Current State:
  Segment A: [{"id":1,"name":"Alice"}{"id":2,"name":"Bob"}...]
                                    ↑
                          _firstSegmentStartIndex = 24

┌─────────────────────────────────────────────────────────────────┐
│ Iteration 3: Read second object                                │
└─────────────────────────────────────────────────────────────────┘

1. helper.Read()
2. _jsonReader.Read() → Returns true (still data in buffer)
3. No need to call MoveNext() yet

┌─────────────────────────────────────────────────────────────────┐
│ Iteration 4: Deserialize second object                         │
└─────────────────────────────────────────────────────────────────┘

1. helper.Deserialise<Person>()
2. Process similar to first object
3. Result: Person { Id=2, Name="Bob" }
4. _firstSegmentStartIndex advances further

┌─────────────────────────────────────────────────────────────────┐
│ Iteration 5: Need more data (buffer boundary)                  │
└─────────────────────────────────────────────────────────────────┘

Suppose third object is split across buffer boundary:

Segment A: [..."Bob"}{"id":3,"na]  ← Partial data
                                 ↑
                         Segment ends here

1. helper.Read() → _jsonReader.Read() returns false (incomplete)
2. MoveNext() called
3. Rent Segment B, link to Segment A
4. stream.Read() fills Segment B: [me":"Charlie"}...]
5. Create ReadOnlySequence spanning [SegmentA + SegmentB]
6. Now _jsonReader.Read() succeeds

Memory:
  [Segment A]→→[Segment B]
   ...3,"na     me":"Ch...

┌─────────────────────────────────────────────────────────────────┐
│ Iteration 6: Cleanup                                           │
└─────────────────────────────────────────────────────────────────┘

After processing third object:
1. Segment A fully consumed → Disposed, returned to pool ♻️
2. Only Segment B remains active
3. Ready for next objects if stream continues
```

---

## 🎓 Key Design Patterns

### 1. **Memory Pooling**
```csharp
Buffer = MemoryPool<byte>.Shared.Rent(size);
```
- Reuses memory instead of allocating new buffers
- Reduces GC pressure significantly
- Critical for high-throughput scenarios

### 2. **ReadOnlySequence<byte>**
```csharp
var data = new ReadOnlySequence<byte>(
    _firstSegment, 
    _firstSegmentStartIndex, 
    _lastSegment, 
    _lastSegmentEndIndex
);
```
- Zero-copy abstraction over discontiguous memory
- Allows Utf8JsonReader to work across segment boundaries
- No need to copy data into contiguous buffer

### 3. **Incremental Disposal**
```csharp
while (_firstSegmentStartIndex > 0 && 
       _firstSegment?.Memory.Length <= _firstSegmentStartIndex)
{
    var currFirstSegment = _firstSegment;
    _firstSegmentStartIndex -= _firstSegment.Memory.Length;
    _firstSegment = (SequenceSegment?)_firstSegment.Next;
    
    if (!_keepBuffers)
        currFirstSegment.Dispose();
}
```
- Release memory as soon as it's consumed
- Maintains minimal memory footprint
- Critical for processing large streams

### 4. **Depth-Based Object Boundary Detection**
```csharp
int depth = 0;
if (TokenType == JsonTokenType.StartObject || 
    TokenType == JsonTokenType.StartArray)
    depth++;

while (depth > 0 && Read())
{
    if (TokenType == JsonTokenType.StartObject || 
        TokenType == JsonTokenType.StartArray)
        depth++;
    else if (TokenType == JsonTokenType.EndObject || 
             TokenType == JsonTokenType.EndArray)
        depth--;
}
```
- Handles nested objects correctly
- Works with arrays and complex structures
- Ensures complete object extraction

---

## ⚠️ Important Considerations

### 1. **Buffer Size Caveat**
```csharp
private readonly int _bufferSize;
// ⚠️ Note: Actual buffers may be larger - don't use for calculations
```

**Why?** `MemoryPool<byte>.Shared.Rent(size)` may return buffers larger than requested. Always use `Buffer.Memory.Length` for actual size.

### 2. **_keepBuffers Flag**
```csharp
_keepBuffers = true;  // During DeserialisePre()
_keepBuffers = false; // After DeserialisePost()
```

**Purpose:** Prevents disposal during depth scanning when we need to keep the entire object in memory for deserialization.

### 3. **_isFinalBlock Logic**
```csharp
_isFinalBlock = _lastSegmentEndIndex < newSegment.Buffer.Memory.Length;
```

**Meaning:** If we read less than the buffer size, we've reached end of stream.

### 4. **ref struct Limitation**
```csharp
public ref struct Utf8JsonStreamReaderHelper
```

**Implications:**
- ✅ Stack-allocated only (better performance)
- ❌ Cannot be stored in fields
- ❌ Cannot be used in async methods
- ❌ Cannot be boxed

---

## 📊 Performance Characteristics

| Operation | Time Complexity | Memory |
|-----------|----------------|--------|
| Read() | O(1) amortized | O(bufferSize) |
| Deserialise<T>() | O(n) where n = object size | O(object size) |
| MoveNext() | O(1) | O(bufferSize) |
| Segment disposal | O(k) where k = consumed segments | Returns to pool |

### Memory Usage Example

```
Stream size: 10 MB
Buffer size: 4 KB
Objects: 1000 small JSON objects

Traditional approach:
  - Load entire 10 MB into memory
  - Memory: 10 MB minimum

Utf8JsonStreamReaderHelper approach:
  - 1-3 segments active at any time (4-12 KB)
  - Memory: ~12 KB maximum
  - 833x more efficient! 🚀
```

---

## 🔧 Usage Example

```csharp
public class Person
{
    public int Id { get; set; }
    public string Name { get; set; }
}

// Usage
using var stream = File.OpenRead("large-data.json");
using var helper = new Utf8JsonStreamReaderHelper(stream, bufferSize: 4096);

var people = new List<Person>();

while (helper.Read())
{
    if (helper.TokenType == JsonTokenType.StartObject)
    {
        var person = helper.Deserialise<Person>();
        if (person != null)
            people.Add(person);
    }
}

// people now contains all Person objects from the stream
// Memory usage stayed minimal throughout!
```

---

## 🎯 Use Cases

### ✅ Perfect For:
- Processing large JSON files that don't fit in memory
- Streaming JSON data from network sources
- Reading JSON logs in real-time
- ETL pipelines with JSON data
- Memory-constrained environments

### ❌ Not Ideal For:
- Small JSON files (< 1MB) - use `JsonSerializer.Deserialize()` directly
- When you need async operations (ref struct limitation)
- When entire document must be in memory anyway
- Random access to JSON elements

---

## 🔍 Debugging Tips

### Visualize Current State
```csharp
// Add this helper method for debugging
private void DumpState()
{
    Console.WriteLine($"First Segment: {_firstSegment != null}");
    Console.WriteLine($"First Index: {_firstSegmentStartIndex}");
    Console.WriteLine($"Last Segment: {_lastSegment != null}");
    Console.WriteLine($"Last Index: {_lastSegmentEndIndex}");
    Console.WriteLine($"Is Final: {_isFinalBlock}");
    Console.WriteLine($"Keep Buffers: {_keepBuffers}");
    Console.WriteLine($"Token: {_jsonReader.TokenType}");
}
```

### Common Issues

**Problem:** Incomplete objects
```
Solution: Check _isFinalBlock before assuming end of data
```

**Problem:** Memory not released
```
Solution: Ensure Dispose() is called, use 'using' statement
```

**Problem:** Unexpected parsing errors
```
Solution: Verify JSON is well-formed, check buffer size is adequate
```

---

## 📚 Related Concepts

### System.Text.Json.Utf8JsonReader
- Low-level, forward-only JSON reader
- Minimal allocations
- Requires `ReadOnlySpan<byte>` or `ReadOnlySequence<byte>`

### System.Buffers.ReadOnlySequence<T>
- Represents a sequence of memory segments
- Zero-copy across discontiguous memory
- Essential for efficient streaming

### System.Buffers.MemoryPool<T>
- Reusable memory buffers
- Reduces GC pressure
- Thread-safe rental/return

---

## 🏆 Summary

`Utf8JsonStreamReaderHelper` is a sophisticated solution for memory-efficient JSON streaming that:

1. **Segments** large streams into manageable chunks
2. **Pools** memory to avoid allocations
3. **Streams** data progressively
4. **Disposes** consumed segments incrementally
5. **Handles** object boundaries correctly with depth tracking

Perfect for production systems dealing with large-scale JSON data! 🚀

---

## 📖 References

- [System.Text.Json Utf8JsonReader](https://learn.microsoft.com/en-us/dotnet/api/system.text.json.utf8jsonreader)
- [ReadOnlySequence<T>](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.readonlysequence-1)
- [MemoryPool<T>](https://learn.microsoft.com/en-us/dotnet/api/system.buffers.memorypool-1)
- [High-performance JSON parsing in .NET](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/how-to)

---

**Document Version:** 1.0  
**Last Updated:** January 2026  
**Author:** Technical Documentation Team

-------------------------------------------------------------------------------------------------------------------
##**Second Approach**

# Utf8JsonStreamReaderHelper — Architectural & Operational Guide

> **Audience**: Senior .NET engineers, performance-focused developers, architects
> **Intent**: Explain *what must always be true* (invariants) **and** *how it actually works* (operational detail), without misleading simplifications.

---

## 1. Problem Statement (Why This Exists)

Parsing JSON from a `Stream` is fundamentally different from parsing JSON from a byte array or string.

When dealing with streams:

* JSON objects **do not align** with read boundaries
* Tokens may be split across buffers
* The stream may be arbitrarily large
* Loading the full payload into memory is often unacceptable

`Utf8JsonStreamReaderHelper` exists to **bridge these constraints** with the guarantees of `System.Text.Json`:

* `Utf8JsonReader` requires a *contiguous logical view* of bytes
* `JsonSerializer.Deserialize<T>` can deserialize **exactly one JSON value**

This helper provides:

* Incremental streaming reads
* Zero-copy logical contiguity via `ReadOnlySequence<byte>`
* Precise extraction of **one complete JSON value at a time**
* Explicit, deterministic memory ownership

---

## 2. Non‑Negotiable Design Invariants

These are the rules this implementation **must never violate**. Everything else exists to uphold them.

### Invariant 1 — The reader must never observe disposed memory

* Any memory referenced by `Utf8JsonReader` **must remain valid** for the duration of its use
* Disposal is deferred whenever an object scan requires multiple passes (`_keepBuffers`)

### Invariant 2 — JSON tokens may span buffer boundaries

* No assumption is made that a token fits in a single buffer
* Logical contiguity is provided by `ReadOnlySequence<byte>`, not copying

### Invariant 3 — Deserialization must operate on a *single JSON value*

* `JsonSerializer.Deserialize<T>` cannot stop mid-stream
* A complete object/array must be isolated before deserialization

### Invariant 4 — Memory usage scales with the *largest JSON value*, not the stream

* Small objects → minimal memory
* One very large object → fully buffered (by necessity)
* This is unavoidable and correct

### Invariant 5 — Reader state must remain continuous across buffer growth

* `Utf8JsonReader` is recreated as data grows
* Parsing state continuity is preserved via `CurrentState`

---

## 3. High-Level Architecture

```
Stream (file / network)
        ↓
[Segment A] → [Segment B] → [Segment C]
        ↓        ↓        ↓
         └──── ReadOnlySequence<byte> ────┘
                         ↓
                Utf8JsonReader (stateful)
                         ↓
           Isolated Slice → Deserialize<T>()
```

Key idea:

> **Physical memory is segmented; logical JSON is contiguous.**

---

## 4. Why This Is a `ref struct`

```csharp
public ref struct Utf8JsonStreamReaderHelper
```

This is an intentional safety constraint:

* Prevents heap allocation
* Prevents boxing
* Prevents async/await misuse
* Guarantees stack-bound lifetime

This aligns with the lifetime rules of `Utf8JsonReader` and pooled memory.

---

## 5. Segment Model and Active Window

### Segment Chain

Each `SequenceSegment`:

* Owns a pooled buffer (`IMemoryOwner<byte>`)
* Is linked bidirectionally (`Previous` / `Next`)
* Participates in a `ReadOnlySequence<byte>`

```
[ Segment A ] → [ Segment B ] → [ Segment C ]
```

### Active Window

Only a subset of the chain is visible to the reader:

```
Segment A        Segment B        Segment C
|--consumed--|--active bytes--|--free space--|
      ↑                          ↑
firstSegmentStartIndex     lastSegmentEndIndex
```

Everything outside this window is either:

* Already consumed
* Not yet read

---

## 6. `Read()` — Forward Progress Guarantee

```csharp
public bool Read()
```

Purpose:

> Ensure the reader advances **or** conclusively determines end-of-stream.

Control flow:

1. Attempt `_jsonReader.Read()`
2. If successful → return `true`
3. If unsuccessful and final block reached → return `false`
4. Otherwise → `MoveNext()` and retry

This loop is what allows tokens to span buffers safely.

---

## 7. `MoveNext()` — Buffer Growth and Disposal

This is the most critical method.

### Responsibilities

1. Advance past consumed bytes
2. Dispose fully-consumed segments (unless prohibited)
3. Rent a new pooled buffer
4. Read more bytes from the stream
5. Rebuild `ReadOnlySequence<byte>`
6. Recreate `Utf8JsonReader` with preserved state

### Why recreate the reader?

`Utf8JsonReader`:

* Is a struct
* Does not support appending input
* Requires a new instance when data grows

State continuity is preserved via:

```csharp
_jsonReader.CurrentState
```

---

## 8. Object Boundary Detection (`DeserialisePre`)

### Why this is unavoidable

Streams may contain:

```
{...}{...}{...}
```

But `JsonSerializer.Deserialize<T>` requires **exactly one JSON value**.

### Mechanism

1. Capture the token start index
2. Enable `_keepBuffers`
3. Track nesting depth

```
StartObject / StartArray → depth++
EndObject   / EndArray   → depth--
```

4. Stop when depth returns to zero

This guarantees:

* Complete object isolation
* Correct handling of nested structures

---

## 9. Isolation and Deserialization (`Deserialise<T>`)

```csharp
var slice = sequence.Slice(tokenStart, reader.Position);
var localReader = new Utf8JsonReader(slice, true, default);
JsonSerializer.Deserialize<T>(ref localReader);
```

Key properties:

* No copying
* No re-parsing
* No shared state
* Serializer sees *only* the object

---

## 10. Post‑Deserialization Cleanup (`DeserialisePost`)

After a value is deserialized:

* Consumed segments are disposed
* Start index is advanced
* Reader is rebuilt if the visible window changed

This restores steady-state streaming behavior.

---

## 11. `SequenceSegment` — Memory Ownership

Purpose:

* Encapsulate pooled memory
* Participate in `ReadOnlySequence<byte>`
* Provide deterministic disposal

### Disposal Strategy

Disposal is **iterative**, not recursive:

```
C → B → A → null
```

This avoids stack overflow and guarantees full cleanup.

---

## 12. Memory Behavior (Important Clarification)

Memory usage characteristics:

* Proportional to the **largest JSON value currently being processed**
* Independent of total stream size
* Temporarily elevated during `_keepBuffers = true`

This is not an optimization artifact — it is a correctness requirement.

---

## 13. When This Design Is Appropriate

### Ideal Use Cases

* Large JSON streams
* Concatenated JSON values
* ETL / ingestion pipelines
* Memory-constrained environments
* High-throughput systems

### When Not to Use

* Small, bounded JSON payloads
* Async parsing requirements
* Scenarios needing random access

---

## 14. Final Summary

`Utf8JsonStreamReaderHelper` is **infrastructure-level code** that:

* Treats JSON parsing as a streaming problem
* Maintains strict memory ownership
* Preserves reader correctness across buffer boundaries
* Trades simplicity for determinism and performance

This design is complex because the problem is complex — and the implementation correctly reflects that reality.

---

*End of authoritative combined document*
