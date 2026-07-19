# RapidJSON C++ Primer

RapidJSON is a fast, header-only C++ library for parsing and generating JSON. It is lower-level and more performance-oriented than libraries like `nlohmann/json`, but it requires you to understand its allocator/ownership model.

Namespace:

```cpp
namespace rj = rapidjson;
```

---

## 1. What RapidJSON Gives You

RapidJSON has two main APIs:

| API | Best for |
|---|---|
| **DOM API** | Parse JSON into a tree (`Document` / `Value`), inspect/modify it, then serialize it. |
| **SAX API** | Stream JSON events through a handler. Low memory, good for large files or filtering/validation. |

Key traits:

- Header-only.
- Very fast.
- Supports UTF-8, UTF-16, UTF-32, transcoding.
- Supports both compact and pretty JSON writing.
- Uses custom allocators.
- Does **not** throw exceptions by default.
- Uses assertions for programmer errors, e.g. accessing a string as an int.

---

## 2. Setup

RapidJSON is header-only. Add its `include/` directory to your compiler include path.

Example:

```sh
g++ -std=c++17 -I/path/to/rapidjson/include main.cpp -o app
```

Common includes:

```cpp
#include <rapidjson/document.h>       // DOM parsing
#include <rapidjson/writer.h>         // compact JSON writer
#include <rapidjson/prettywriter.h>   // pretty JSON writer
#include <rapidjson/stringbuffer.h>   // output to string
#include <rapidjson/error/en.h>       // parse error messages
#include <rapidjson/reader.h>         // SAX parsing
```

Optional but useful:

```cpp
#define RAPIDJSON_HAS_STDSTRING 1
#include <rapidjson/document.h>
```

This enables some `std::string` convenience overloads.

---

## 3. Core Types

```cpp
rapidjson::Document doc;
rapidjson::Value value;
```

Important types:

| Type | Meaning |
|---|---|
| `Document` | Owns a JSON tree and an allocator. Inherits from `Value`. |
| `Value` | A JSON value: null, bool, number, string, array, object. |
| `Value::AllocatorType` | Allocator used for strings/arrays/objects. |
| `StringBuffer` | Output buffer for writing JSON to a string. |
| `Writer<StringBuffer>` | Compact JSON serializer. |
| `PrettyWriter<StringBuffer>` | Human-readable JSON serializer. |
| `Reader` | SAX-style JSON parser. |
| `StringStream` | Input stream from a C string. |
| `FileReadStream` / `FileWriteStream` | File input/output streams. |
| `IStreamWrapper` / `OStreamWrapper` | C++ `std::istream` / `std::ostream` wrappers. |

JSON value types:

```cpp
rapidjson::kNullType
rapidjson::kFalseType
rapidjson::kTrueType
rapidjson::kObjectType
rapidjson::kArrayType
rapidjson::kStringType
rapidjson::kNumberType
```

---

# 4. Parsing JSON with the DOM API

Basic example:

```cpp
#include <rapidjson/document.h>
#include <rapidjson/error/en.h>
#include <iostream>

int main() {
    const char* json = R"({
        "name": "Alice",
        "age": 30,
        "active": true,
        "score": 92.5,
        "tags": ["admin", "dev"]
    })";

    rapidjson::Document doc;
    doc.Parse(json);

    if (doc.HasParseError()) {
        std::cerr << "Parse error: "
                  << rapidjson::GetParseError_En(doc.GetParseError())
                  << " at offset "
                  << doc.GetErrorOffset()
                  << '\n';
        return 1;
    }

    if (!doc.IsObject()) {
        std::cerr << "Root is not an object\n";
        return 1;
    }

    if (doc.HasMember("name") && doc["name"].IsString()) {
        std::cout << "name: " << doc["name"].GetString() << '\n';
    }

    if (doc.HasMember("age") && doc["age"].IsInt()) {
        std::cout << "age: " << doc["age"].GetInt() << '\n';
    }

    if (doc.HasMember("active") && doc["active"].IsBool()) {
        std::cout << "active: " << std::boolalpha << doc["active"].GetBool() << '\n';
    }

    if (doc.HasMember("score") && doc["score"].IsNumber()) {
        std::cout << "score: " << doc["score"].GetDouble() << '\n';
    }

    if (doc.HasMember("tags") && doc["tags"].IsArray()) {
        const auto& tags = doc["tags"];

        for (rapidjson::SizeType i = 0; i < tags.Size(); ++i) {
            if (tags[i].IsString()) {
                std::cout << "tag: " << tags[i].GetString() << '\n';
            }
        }

        // Range-based form:
        for (const auto& tag : tags.GetArray()) {
            if (tag.IsString()) {
                std::cout << "tag: " << tag.GetString() << '\n';
            }
        }
    }

    // Iterate object members:
    for (const auto& member : doc.GetObject()) {
        std::cout << "member: " << member.name.GetString() << '\n';
    }
}
```

---

## 5. Safe Access Patterns

`operator[]` is convenient but can assert if the member does not exist or has the wrong type.

Prefer explicit checks:

```cpp
if (doc.HasMember("name") && doc["name"].IsString()) {
    const char* name = doc["name"].GetString();
}
```

Or use `FindMember`:

```cpp
auto it = doc.FindMember("name");

if (it != doc.MemberEnd() && it->value.IsString()) {
    const char* name = it->value.GetString();
}
```

A reusable helper:

```cpp
bool GetStringMember(const rapidjson::Value& obj,
                     const char* key,
                     std::string& out) {
    auto it = obj.FindMember(key);
    if (it != obj.MemberEnd() && it->value.IsString()) {
        out.assign(it->value.GetString(), it->value.GetStringLength());
        return true;
    }
    return false;
}
```

Use similar helpers for ints, bools, doubles, arrays, etc.

---

## 6. Type Checks

Common checks:

```cpp
value.IsNull();
value.IsBool();
value.IsTrue();
value.IsFalse();
value.IsInt();
value.IsUint();
value.IsInt64();
value.IsUint64();
value.IsDouble();
value.IsNumber();
value.IsString();
value.IsArray();
value.IsObject();
```

JSON has one number type, but RapidJSON stores numbers internally as one of:

- `int`
- `unsigned`
- `int64_t`
- `uint64_t`
- `double`

So check carefully:

```cpp
if (v.IsInt()) {
    int x = v.GetInt();
} else if (v.IsDouble()) {
    double x = v.GetDouble();
}
```

`IsNumber()` is true for any numeric representation.

---

## 7. Building JSON

Example:

```cpp
#include <rapidjson/document.h>
#include <rapidjson/stringbuffer.h>
#include <rapidjson/writer.h>
#include <rapidjson/prettywriter.h>
#include <iostream>

int main() {
    rapidjson::Document doc;
    doc.SetObject();

    auto& alloc = doc.GetAllocator();

    doc.AddMember("name", rapidjson::Value("Alice", alloc).Move(), alloc);
    doc.AddMember("age", rapidjson::Value(30).Move(), alloc);
    doc.AddMember("active", rapidjson::Value(true).Move(), alloc);

    rapidjson::Value tags(rapidjson::kArrayType);
    tags.PushBack(rapidjson::Value("admin", alloc).Move(), alloc);
    tags.PushBack(rapidjson::Value("dev", alloc).Move(), alloc);

    doc.AddMember("tags", tags, alloc);

    rapidjson::Value address(rapidjson::kObjectType);
    address.AddMember("city", rapidjson::Value("Paris", alloc).Move(), alloc);
    address.AddMember("zip", rapidjson::Value("75001", alloc).Move(), alloc);

    doc.AddMember("address", address, alloc);

    // Compact output
    rapidjson::StringBuffer sb;
    rapidjson::Writer<rapidjson::StringBuffer> writer(sb);
    doc.Accept(writer);

    std::cout << sb.GetString() << '\n';

    // Pretty output
    sb.Clear();

    rapidjson::PrettyWriter<rapidjson::StringBuffer> pretty(sb);
    pretty.SetIndent(' ', 2);
    doc.Accept(pretty);

    std::cout << sb.GetString() << '\n';
}
```

Output compact:

```json
{"name":"Alice","age":30,"active":true,"tags":["admin","dev"],"address":{"city":"Paris","zip":"75001"}}
```

Pretty output:

```json
{
  "name": "Alice",
  "age": 30,
  "active": true,
  "tags": [
    "admin",
    "dev"
  ],
  "address": {
    "city": "Paris",
    "zip": "75001"
  }
}
```

---

## 8. Modifying an Existing Document

```cpp
#include <rapidjson/document.h>
#include <rapidjson/stringbuffer.h>
#include <rapidjson/writer.h>
#include <rapidjson/error/en.h>
#include <iostream>

int main() {
    const char* json = R"({
        "user": "alice",
        "scores": [10, 20, 30]
    })";

    rapidjson::Document doc;
    doc.Parse(json);

    if (doc.HasParseError()) {
        std::cerr << rapidjson::GetParseError_En(doc.GetParseError()) << '\n';
        return 1;
    }

    auto& alloc = doc.GetAllocator();

    // Modify existing string
    auto it = doc.FindMember("user");
    if (it != doc.MemberEnd() && it->value.IsString()) {
        it->value.SetString("bob", alloc);
    }

    // Append to array
    auto scores = doc.FindMember("scores");
    if (scores != doc.MemberEnd() && scores->value.IsArray()) {
        scores->value.PushBack(rapidjson::Value(40).Move(), alloc);
    }

    // Add a new member
    doc.AddMember("updated", rapidjson::Value(true).Move(), alloc);

    rapidjson::StringBuffer sb;
    rapidjson::Writer<rapidjson::StringBuffer> writer(sb);
    doc.Accept(writer);

    std::cout << sb.GetString() << '\n';
}
```

Important:

- `AddMember()` does **not** replace an existing key.
- If the key already exists, it adds another member with the same name.
- JSON objects in RapidJSON preserve insertion order.
- Object lookup is linear, not hash-based.

To replace a member:

```cpp
auto it = doc.FindMember("user");

if (it != doc.MemberEnd()) {
    it->value.SetString("bob", alloc);
} else {
    doc.AddMember("user", rapidjson::Value("bob", alloc).Move(), alloc);
}
```

---

## 9. Allocators, Ownership, and Move Semantics

This is the most important RapidJSON concept.

RapidJSON values do not manage memory like `std::vector` or `std::string`. They use an allocator, usually owned by a `Document`.

```cpp
rapidjson::Document doc;
auto& alloc = doc.GetAllocator();
```

When you create strings, arrays, or objects, you often need the allocator:

```cpp
rapidjson::Value name("Alice", alloc); // copies string into allocator
```

### Move semantics

RapidJSON values are usually moved, not copied.

```cpp
rapidjson::Value a(123);
rapidjson::Value b;

b = a; // moves a into b; a becomes Null
```

To explicitly move:

```cpp
doc.AddMember("x", rapidjson::Value(123).Move(), alloc);
```

After a value is moved, the source becomes `Null`.

### Copying

Use `CopyFrom` or the copy constructor with an allocator:

```cpp
rapidjson::Value a("hello", alloc);

rapidjson::Value b;
b.CopyFrom(a, alloc);

rapidjson::Value c(a, alloc);
```

### Allocator lifetime

The allocator must outlive any values using it.

Bad:

```cpp
rapidjson::Value* leaky = nullptr;

{
    rapidjson::Document doc;
    auto& alloc = doc.GetAllocator();

    rapidjson::Value v("hello", alloc);
    leaky = &v;
}

// doc and allocator are destroyed.
// leaky now points to invalid memory.
```

### Moving between documents

Be careful moving values between documents with different allocators.

Usually prefer copying into the target allocator:

```cpp
rapidjson::Document source;
source.Parse(R"({"x": 1})");

rapidjson::Document target;
target.SetObject();

auto& targetAlloc = target.GetAllocator();

rapidjson::Value copied;
copied.CopyFrom(source["x"], targetAlloc);

target.AddMember("x", copied, targetAlloc);
```

---

## 10. Strings

RapidJSON strings can be either:

1. Non-copying references.
2. Copied into an allocator.

### Non-copying string literal

```cpp
rapidjson::Value v("literal");
```

This is safe for string literals because they live for the program lifetime.

### Copying a runtime string

```cpp
std::string s = get_name();

rapidjson::Value v;
v.SetString(s.c_str(), static_cast<rapidjson::SizeType>(s.size()), alloc);
```

Or with `RAPIDJSON_HAS_STDSTRING`:

```cpp
rapidjson::Value v(s, alloc);
```

### Reading strings

```cpp
const char* str = value.GetString();
rapidjson::SizeType len = value.GetStringLength();
```

`GetString()` returns a pointer to internal storage. Do not delete it.

If the JSON string may contain embedded null characters, use `GetStringLength()`.

---

## 11. Arrays

Create:

```cpp
rapidjson::Value arr(rapidjson::kArrayType);

arr.PushBack(1, alloc);
arr.PushBack(2, alloc);
arr.PushBack(3, alloc);
```

Read:

```cpp
if (doc.HasMember("items") && doc["items"].IsArray()) {
    const auto& arr = doc["items"];

    for (rapidjson::SizeType i = 0; i < arr.Size(); ++i) {
        if (arr[i].IsInt()) {
            std::cout << arr[i].GetInt() << '\n';
        }
    }
}
```

Modify:

```cpp
arr[0].SetInt(100);
arr.PopBack();
arr.Clear();
```

Range-based iteration:

```cpp
for (auto& item : arr.GetArray()) {
    // ...
}
```

---

## 12. Objects

Create:

```cpp
rapidjson::Value obj(rapidjson::kObjectType);

obj.AddMember("id", rapidjson::Value(1).Move(), alloc);
obj.AddMember("name", rapidjson::Value("Alice", alloc).Move(), alloc);
```

Read:

```cpp
if (doc.IsObject()) {
    for (auto& member : doc.GetObject()) {
        const char* key = member.name.GetString();
        rapidjson::Value& value = member.value;

        std::cout << key << '\n';
    }
}
```

Find:

```cpp
auto it = doc.FindMember("id");

if (it != doc.MemberEnd()) {
    if (it->value.IsInt()) {
        int id = it->value.GetInt();
    }
}
```

Remove:

```cpp
auto it = doc.FindMember("temporary");

if (it != doc.MemberEnd()) {
    doc.EraseMember(it);
}
```

---

## 13. Writing JSON to a String

Compact:

```cpp
rapidjson::StringBuffer sb;
rapidjson::Writer<rapidjson::StringBuffer> writer(sb);

doc.Accept(writer);

std::string json = sb.GetString();
```

Pretty:

```cpp
rapidjson::StringBuffer sb;
rapidjson::PrettyWriter<rapidjson::StringBuffer> writer(sb);

writer.SetIndent(' ', 2);
doc.Accept(writer);

std::string json = sb.GetString();
```

If your JSON may contain embedded nulls, use:

```cpp
std::string json(sb.GetString(), sb.GetSize());
```

---

## 14. SAX API: Streaming Parse

The SAX API does not build a DOM tree. It calls handler methods as it encounters JSON events.

Useful for:

- Large files.
- Low memory usage.
- Validation.
- Filtering.
- Counting.
- Transforming JSON while reading.

Example: count object keys.

```cpp
#include <rapidjson/reader.h>
#include <rapidjson/stringbuffer.h>
#include <rapidjson/error/en.h>
#include <iostream>

struct KeyCounter : public rapidjson::BaseReaderHandler<rapidjson::UTF8<>, KeyCounter> {
    int keys = 0;

    bool Key(const char* str, rapidjson::SizeType length, bool copy) {
        ++keys;
        return true;
    }
};

int main() {
    const char* json = R"({
        "a": 1,
        "b": {
            "c": 2
        }
    })";

    rapidjson::Reader reader;
    rapidjson::StringStream ss(json);

    KeyCounter handler;

    rapidjson::ParseResult result = reader.Parse(ss, handler);

    if (result.IsError()) {
        std::cerr << rapidjson::GetParseError_En(result.Code()) << '\n';
        return 1;
    }

    std::cout << "keys: " << handler.keys << '\n';
}
```

Handler methods include:

```cpp
bool Null();
bool Bool(bool b);
bool Int(int i);
bool Uint(unsigned u);
bool Int64(int64_t i);
bool Uint64(uint64_t u);
bool Double(double d);
bool RawNumber(const char* str, rapidjson::SizeType length, bool copy);
bool String(const char* str, rapidjson::SizeType length, bool copy);
bool StartObject();
bool Key(const char* str, rapidjson::SizeType length, bool copy);
bool EndObject(rapidjson::SizeType memberCount);
bool StartArray();
bool EndArray(rapidjson::SizeType elementCount);
```

Returning `false` stops parsing.

---

## 15. SAX Parsing into a Writer

You can parse and immediately write JSON:

```cpp
#include <rapidjson/reader.h>
#include <rapidjson/writer.h>
#include <rapidjson/stringbuffer.h>

int main() {
    const char* json = R"({"a":1,"b":[true,null,"x"]})";

    rapidjson::Reader reader;
    rapidjson::StringStream input(json);

    rapidjson::StringBuffer output;
    rapidjson::Writer<rapidjson::StringBuffer> writer(output);

    reader.Parse(input, writer);

    std::string normalized = output.GetString();
}
```

This is useful for normalization, validation, or lightweight transformation.

---

## 16. File I/O

Read from file:

```cpp
#include <rapidjson/document.h>
#include <rapidjson/filereadstream.h>
#include <cstdio>

bool ReadJsonFile(const char* path, rapidjson::Document& doc) {
    FILE* fp = std::fopen(path, "rb");
    if (!fp) {
        return false;
    }

    char buffer[65536];
    rapidjson::FileReadStream is(fp, buffer, sizeof(buffer));

    doc.ParseStream(is);

    std::fclose(fp);

    return !doc.HasParseError();
}
```

Write to file:

```cpp
#include <rapidjson/document.h>
#include <rapidjson/filewritestream.h>
#include <rapidjson/writer.h>
#include <cstdio>

bool WriteJsonFile(const char* path, const rapidjson::Document& doc) {
    FILE* fp = std::fopen(path, "wb");
    if (!fp) {
        return false;
    }

    char buffer[65536];
    rapidjson::FileWriteStream os(fp, buffer, sizeof(buffer));

    rapidjson::Writer<rapidjson::FileWriteStream> writer(os);
    doc.Accept(writer);

    std::fclose(fp);
    return true;
}
```

Use binary mode, `"rb"` and `"wb"`, especially on Windows.

---

## 17. C++ Stream I/O

RapidJSON can wrap `std::istream` and `std::ostream`.

```cpp
#include <rapidjson/document.h>
#include <rapidjson/istreamwrapper.h>
#include <rapidjson/ostreamwrapper.h>
#include <rapidjson/writer.h>
#include <fstream>

void ReadFromStdStream(const char* path, rapidjson::Document& doc) {
    std::ifstream ifs(path, std::ios::binary);
    rapidjson::IStreamWrapper isw(ifs);
    doc.ParseStream(isw);
}

void WriteToStdStream(const char* path, const rapidjson::Document& doc) {
    std::ofstream ofs(path, std::ios::binary);
    rapidjson::OStreamWrapper osw(ofs);

    rapidjson::Writer<rapidjson::OStreamWrapper> writer(osw);
    doc.Accept(writer);
}
```

---

## 18. Error Handling

Parse errors are reported through:

```cpp
doc.HasParseError();
doc.GetParseError();
doc.GetErrorOffset();
```

Human-readable message:

```cpp
#include <rapidjson/error/en.h>

const char* msg = rapidjson::GetParseError_En(doc.GetParseError());
```

Example:

```cpp
doc.Parse(json);

if (doc.HasParseError()) {
    std::cerr << "Error: "
              << rapidjson::GetParseError_En(doc.GetParseError())
              << " at offset "
              << doc.GetErrorOffset()
              << '\n';
}
```

For SAX:

```cpp
rapidjson::ParseResult result = reader.Parse(ss, handler);

if (result.IsError()) {
    std::cerr << rapidjson::GetParseError_En(result.Code()) << '\n';
}
```

---

## 19. Parse Flags

RapidJSON supports optional parse flags.

```cpp
doc.Parse<rapidjson::kParseFullPrecisionFlag>(json);
```

Combine flags:

```cpp
doc.Parse<
    rapidjson::kParseFullPrecisionFlag |
    rapidjson::kParseValidateEncodingFlag
>(json);
```

Common flags:

| Flag | Meaning |
|---|---|
| `kParseDefaultFlags` | Default parsing behavior. |
| `kParseInsituFlag` | In-situ parsing. Modifies input buffer. |
| `kParseValidateEncodingFlag` | Validate input encoding. |
| `kParseFullPrecisionFlag` | Parse doubles with full precision. |
| `kParseNumbersAsStringsFlag` | Preserve numbers as strings. |
| `kParseTrailingCommasFlag` | Allow trailing commas. |
| `kParseNanAndInfFlag` | Allow `NaN` and `Inf`. |

Example:

```cpp
char json[] = R"({"name":"Alice"})";

rapidjson::Document doc;
doc.ParseInsitu(json);
```

In-situ parsing modifies the input buffer and can be faster, but the buffer must remain valid while you use the parsed strings.

---

## 20. In-Situ Parsing

Normal parsing copies strings into the document allocator.

In-situ parsing unescapes strings directly inside the input buffer.

```cpp
char json[] = R"({"name":"Alice"})";

rapidjson::Document doc;
doc.ParseInsitu(json);

std::cout << doc["name"].GetString() << '\n';
```

Important:

- Input must be mutable.
- The buffer must stay alive as long as the document uses its strings.
- Good for performance.
- Dangerous if you free or modify the buffer too early.

---

## 21. Numbers and Precision

JSON has one number type. RapidJSON chooses an internal representation.

```cpp
value.IsInt();
value.IsUint();
value.IsInt64();
value.IsUint64();
value.IsDouble();
value.IsNumber();
```

Reading:

```cpp
if (value.IsInt()) {
    int x = value.GetInt();
}

if (value.IsInt64()) {
    int64_t x = value.GetInt64();
}

if (value.IsDouble()) {
    double x = value.GetDouble();
}
```

If you need exact decimal preservation, parse numbers as strings:

```cpp
doc.Parse<rapidjson::kParseNumbersAsStringsFlag>(json);
```

Then handle numeric parsing yourself, possibly with a decimal library.

---

## 22. Common Pitfalls

### 1. Assuming `operator[]` is safe

This may assert:

```cpp
int age = doc["age"].GetInt();
```

Safer:

```cpp
auto it = doc.FindMember("age");

if (it != doc.MemberEnd() && it->value.IsInt()) {
    int age = it->value.GetInt();
}
```

---

### 2. Forgetting to check types

This is dangerous:

```cpp
int x = value.GetInt();
```

If `value` is not an int, RapidJSON may assert.

Always check:

```cpp
if (value.IsInt()) {
    int x = value.GetInt();
}
```

---

### 3. Forgetting the allocator

This will not compile or will not behave as expected:

```cpp
rapidjson::Value v("hello");
doc.AddMember("greeting", v, alloc);
```

For runtime strings, copy into allocator:

```cpp
std::string s = "hello";

rapidjson::Value v;
v.SetString(s.c_str(), static_cast<rapidjson::SizeType>(s.size()), alloc);

doc.AddMember("greeting", v, alloc);
```

---

### 4. Using moved values

After moving, the source becomes `Null`.

```cpp
rapidjson::Value v("hello", alloc);

doc.AddMember("greeting", v, alloc);

// v is now moved-from and should not be reused.
```

If you need a copy:

```cpp
rapidjson::Value copy;
copy.CopyFrom(v, alloc);
```

---

### 5. Letting the allocator die

Bad:

```cpp
rapidjson::Value* p = nullptr;

{
    rapidjson::Document doc;
    auto& alloc = doc.GetAllocator();

    rapidjson::Value v("hello", alloc);
    p = &v;
}

// p is dangling.
```

---

### 6. Assuming object lookup is fast

RapidJSON objects are not hash maps.

`FindMember()` is linear in the number of members.

For repeated access, cache iterators or build your own index.

---

### 7. Assuming `AddMember` replaces keys

It does not.

```cpp
doc.AddMember("x", 1, alloc);
doc.AddMember("x", 2, alloc);
```

Now there are two `"x"` members.

To replace:

```cpp
auto it = doc.FindMember("x");

if (it != doc.MemberEnd()) {
    it->value.SetInt(2);
} else {
    doc.AddMember("x", 2, alloc);
}
```

---

### 8. Ignoring assertions in release builds

RapidJSON uses assertions for programmer errors.

If assertions are disabled, invalid access may become undefined behavior.

For production code, explicitly check types and members.

---

## 23. Exceptions

By default, RapidJSON does not throw exceptions.

Parse errors are returned through error codes.

Programmer errors use assertions.

You can redefine `RAPIDJSON_ASSERT` to throw, but do it consistently before including RapidJSON headers.

Example:

```cpp
#include <stdexcept>

#define RAPIDJSON_ASSERT(x) \
    do { \
        if (!(x)) throw std::runtime_error("RapidJSON assertion failed: " #x); \
    } while (0)

#include <rapidjson/document.h>
```

Use with care.

---

## 24. JSON Pointer

RapidJSON supports RFC 6901 JSON Pointer.

```cpp
#include <rapidjson/document.h>
#include <rapidjson/pointer.h>

rapidjson::Document doc;
doc.Parse(R"({"address":{"city":"Paris"}})");

rapidjson::Value* city = rapidjson::Pointer("/address/city").Get(doc);

if (city && city->IsString()) {
    std::cout << city->GetString() << '\n';
}
```

Useful for accessing nested values without many checks.

---

## 25. JSON Schema Validation

RapidJSON includes JSON Schema support.

```cpp
#include <rapidjson/document.h>
#include <rapidjson/schema.h>

rapidjson::Document schemaJson;
schemaJson.Parse(R"({
    "type": "object",
    "properties": {
        "name": { "type": "string" },
        "age": { "type": "integer", "minimum": 0 }
    },
    "required": ["name"]
})");

rapidjson::SchemaDocument schema(schemaJson);
rapidjson::SchemaValidator validator(schema);

rapidjson::Document doc;
doc.Parse(R"({"name":"Alice","age":30})");

if (!doc.Accept(validator)) {
    // Validation failed.
}
```

This is more advanced but useful for validating untrusted input.

---

## 26. Encodings

RapidJSON is encoding-aware.

Default DOM type is UTF-8:

```cpp
rapidjson::Document doc; // UTF-8 by default
```

You can use other encodings:

```cpp
rapidjson::GenericDocument<rapidjson::UTF16<>> doc16;
```

For files with BOMs or mixed encodings, use encoded stream wrappers such as:

```cpp
rapidjson::AutoUTFInputStream
rapidjson::EncodedInputStream
rapidjson::EncodedOutputStream
```

For most applications, UTF-8 is what you want.

---

## 27. Performance Tips

Use the DOM API when:

- The JSON fits comfortably in memory.
- You need random access.
- You need to modify and reserialize.

Use the SAX API when:

- The JSON is large.
- You only need one pass.
- You want low memory usage.
- You are filtering, counting, validating, or transforming.

Other tips:

- Use `Writer` for production output.
- Use `PrettyWriter` only for debugging.
- Use in-situ parsing if safe.
- Avoid unnecessary copies.
- Use the same allocator for related values.
- Cache object member iterators if doing repeated lookups.
- Use `kParseFullPrecisionFlag` only when needed.
- For exact decimal numbers, store or parse them as strings.

---

## 28. Minimal Cheat Sheet

Parse:

```cpp
rapidjson::Document doc;
doc.Parse(json);

if (doc.HasParseError()) {
    // handle error
}
```

Read:

```cpp
if (doc.HasMember("name") && doc["name"].IsString()) {
    const char* name = doc["name"].GetString();
}
```

Array:

```cpp
const auto& arr = doc["items"];

for (rapidjson::SizeType i = 0; i < arr.Size(); ++i) {
    // arr[i]
}
```

Object:

```cpp
for (const auto& m : doc.GetObject()) {
    const char* key = m.name.GetString();
    const rapidjson::Value& value = m.value;
}
```

Build:

```cpp
rapidjson::Document doc;
doc.SetObject();

auto& alloc = doc.GetAllocator();

doc.AddMember("name", rapidjson::Value("Alice", alloc).Move(), alloc);
doc.AddMember("age", rapidjson::Value(30).Move(), alloc);
```

Array:

```cpp
rapidjson::Value arr(rapidjson::kArrayType);

arr.PushBack(1, alloc);
arr.PushBack(2, alloc);

doc.AddMember("numbers", arr, alloc);
```

Serialize:

```cpp
rapidjson::StringBuffer sb;
rapidjson::Writer<rapidjson::StringBuffer> writer(sb);

doc.Accept(writer);

std::string json = sb.GetString();
```

Pretty serialize:

```cpp
rapidjson::StringBuffer sb;
rapidjson::PrettyWriter<rapidjson::StringBuffer> writer(sb);

writer.SetIndent(' ', 2);
doc.Accept(writer);

std::string json = sb.GetString();
```

---

## 29. Mental Model

Think of RapidJSON like this:

```text
Document
  owns Allocator
  is a Value
    Value can be:
      null
      bool
      number
      string
      array
      object
```

Values are moved by default.

Strings and containers usually allocate through an allocator.

The allocator must outlive the values using it.

Parse errors are explicit.

Type errors are usually assertion/programmer errors.

---

## 30. When to Use RapidJSON

Use RapidJSON when you need:

- High performance.
- Low memory overhead.
- SAX streaming.
- Fine-grained control.
- JSON parsing/generation in C++ without heavy dependencies.

Consider a higher-level library like `nlohmann/json` when you want:

- Simpler syntax.
- Easy conversion to/from C++ structs.
- STL-like containers.
- Less manual memory/ownership thinking.

RapidJSON is powerful, but its allocator and move semantics take some getting used to. Once you understand those, it is one of the most efficient JSON libraries available in C++.
