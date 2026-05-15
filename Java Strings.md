# Java Strings — Complete Deep Dive

---

## Table of Contents

1. [Theory — Internals & JVM Model](#1-theory--internals--jvm-model)
2. [String Creation & The String Pool](#2-string-creation--the-string-pool)
3. [Immutability — Why & How](#3-immutability--why--how)
4. [String Methods — Full Reference with Code](#4-string-methods--full-reference-with-code)
5. [StringBuilder vs StringBuffer vs String](#5-stringbuilder-vs-stringbuffer-vs-string)
6. [String Comparison Deep Dive](#6-string-comparison-deep-dive)
7. [Encoding, Chars, and Unicode](#7-encoding-chars-and-unicode)
8. [Regular Expressions with String](#8-regular-expressions-with-string)
9. [Java 8–21 String Enhancements](#9-java-821-string-enhancements)
10. [Performance Patterns & Pitfalls](#10-performance-patterns--pitfalls)
11. [Strings in Collections & Hashing](#11-strings-in-collections--hashing)
12. [Common Coding Problems & Solutions](#12-common-coding-problems--solutions)
13. [Interview Questions & Answers](#13-interview-questions--answers)

---

## 1. Theory — Internals & JVM Model

### What is a String in Java?

`String` is a **final class** in `java.lang` that represents a sequence of characters. Unlike primitives, it is an object — but it behaves almost like a primitive due to special JVM treatment.

```
java.lang.Object
    └── java.lang.String  (final, implements Serializable, Comparable<String>, CharSequence)
```

### Internal Representation (Pre Java 9)

Before Java 9, `String` was backed by a `char[]` (each char = 2 bytes, UTF-16).

```java
// Pre-Java 9 internal layout (simplified)
public final class String {
    private final char[] value;   // UTF-16 encoded characters
    private int hash;             // cached hashCode (lazy, default 0)
}
```

A string `"Hello"` consumed: `object header (16 bytes) + char[] object (16 + 5×2 = 26 bytes) = ~56 bytes` on heap.

### Internal Representation — Compact Strings (Java 9+)

Java 9 introduced **Compact Strings** (JEP 254). The internal `char[]` was replaced with `byte[]` plus a `coder` field.

```java
// Java 9+ internal layout (simplified)
public final class String {
    private final byte[] value;   // LATIN-1 (1 byte/char) OR UTF-16 (2 bytes/char)
    private final byte coder;     // 0 = LATIN-1, 1 = UTF-16
    private int hash;
    static final byte LATIN1 = 0;
    static final byte UTF16  = 1;
}
```

**Impact:** ASCII-only strings (most English text) now use **half the memory**. This is transparent — the public API is unchanged.

```java
String s1 = "Hello";   // coder = LATIN1, byte[] = {72, 101, 108, 108, 111}
String s2 = "日本語";  // coder = UTF16,  byte[] = 6 bytes (3 chars × 2)
```

### Memory Layout

```
Stack Frame
┌─────────────────┐
│ String ref (s)  │ ──────────────────────────────────────────────────────►  Heap
└─────────────────┘                                                         ┌──────────────────────────┐
                                                                            │ String object            │
                                                                            │  value → byte[] ref      │ ──► byte[] { 72, 101, 108... }
                                                                            │  coder = 0               │
                                                                            │  hash  = 0               │
                                                                            └──────────────────────────┘

String Pool (part of Heap, in Metaspace pre-Java 7 → Heap since Java 7)
┌───────────────────────────────────────────────┐
│ "Hello" → ref  │  "World" → ref  │  ...       │
└───────────────────────────────────────────────┘
```

---

## 2. String Creation & The String Pool

### Two Ways to Create

```java
// Way 1: String literal — goes through the String Pool
String s1 = "Hello";

// Way 2: new keyword — always creates a new object on heap
String s2 = new String("Hello");
```

### String Pool (String Intern Pool)

The **String Pool** is a special region inside the Java Heap (since Java 7; was in PermGen before that) that stores unique string literals.

**How it works:**

1. JVM scans the pool when a string literal is encountered.
2. If the value already exists → returns the **same reference**.
3. If not → creates a new `String` object in the pool and returns its reference.

```java
String a = "Hello";
String b = "Hello";
String c = new String("Hello");

System.out.println(a == b);          // true  — same pool reference
System.out.println(a == c);          // false — c is on heap, not pool
System.out.println(a.equals(c));     // true  — same content

// intern() — manually puts a string into the pool (or returns existing pool ref)
String d = c.intern();
System.out.println(a == d);          // true  — d now points to pool object
```

### Compile-Time Constant Folding

```java
String s1 = "Hello" + " World";   // compile-time constant → "Hello World" in pool
String s2 = "Hello World";

System.out.println(s1 == s2);      // true — compiler folds the literal concatenation

String part = "World";
String s3 = "Hello " + part;      // runtime concatenation — NOT in pool
System.out.println(s2 == s3);     // false
```

---

## 3. Immutability — Why & How

### What Makes String Immutable?

```java
public final class String {
    private final byte[] value;   // final field — reference cannot change
    // no setter methods exposed
    // byte[] contents are never modified after construction
}
```

Four pillars:
1. Class is `final` → cannot be subclassed
2. `value` array is `private final` → reference cannot be reassigned
3. No mutating methods exposed
4. Internal `byte[]` is not shared (defensive copy on construction where needed)

### Why Immutability?

**1. String Pool is possible only because of immutability.**
If strings were mutable, sharing references in the pool would be dangerous — one holder could change the value for all others.

**2. Thread Safety.**
Immutable objects are inherently thread-safe. Multiple threads can read the same `String` instance without synchronization.

**3. Safe Hash Key.**
`HashMap`, `HashSet` rely on stable `hashCode()`. Since `String` is immutable, its hash never changes after creation — perfect for use as a map key.

**4. Security.**
Class loading, network connections, file paths, and reflection APIs use `String`. Mutable strings could be changed after a security check passes, creating TOCTOU vulnerabilities.

```java
// Security example: if String were mutable (hypothetical danger)
String path = "/safe/dir/file.txt";
if (isAllowed(path)) {
    path.setValue("/etc/passwd");  // attack — not possible with real String
    read(path);
}
```

### The "Illusion" of Mutation

```java
String s = "Hello";
s = s + " World";   // s now points to a NEW object; "Hello" still exists in pool
```

---

## 4. String Methods — Full Reference with Code

### Length & Access

```java
String s = "Hello World";

s.length();                // 11 — number of chars
s.charAt(0);               // 'H'
s.indexOf('o');            // 4 — first occurrence
s.lastIndexOf('o');        // 7 — last occurrence
s.indexOf("World");        // 6 — substring search
s.indexOf('o', 5);         // 7 — search starting from index 5
s.isEmpty();               // false
s.isBlank();               // false (Java 11+) — checks whitespace-only
```

### Substring & Slicing

```java
String s = "Hello World";

s.substring(6);            // "World"
s.substring(0, 5);         // "Hello"  [inclusive, exclusive]
s.subSequence(0, 5);       // "Hello"  returns CharSequence
```

### Comparison

```java
String a = "hello";
String b = "Hello";

a.equals(b);                    // false — case-sensitive
a.equalsIgnoreCase(b);          // true
a.compareTo(b);                 // positive (lowercase > uppercase in ASCII)
a.compareToIgnoreCase(b);       // 0
a.startsWith("hel");            // true
a.endsWith("llo");              // true
a.contains("ell");              // true
a.regionMatches(1, "ellow", 0, 4);   // true — sub-range comparison
```

### Transformation

```java
String s = "  Hello World  ";

s.trim();                   // "Hello World"    — removes leading/trailing ASCII whitespace
s.strip();                  // "Hello World"    — Unicode-aware (Java 11+)
s.stripLeading();           // "Hello World  "  (Java 11+)
s.stripTrailing();          // "  Hello World"  (Java 11+)
s.toLowerCase();            // "  hello world  "
s.toUpperCase();            // "  HELLO WORLD  "
s.replace('l', 'r');        // "  Herro Worrd  "
s.replace("World", "Java"); // "  Hello Java  "
s.replaceAll("\\s+", "-");  // "--Hello-World--"  (regex)
s.replaceFirst("\\s+", "-");// "-Hello World  "   (regex, first match)
```

### Split & Join

```java
// split
String csv = "a,b,c,d";
String[] parts = csv.split(",");           // ["a","b","c","d"]
String[] parts2 = csv.split(",", 2);       // ["a","b,c,d"] — limit=2
String[] parts3 = "a::b".split(":", -1);  // ["a","","b"] — negative limit keeps trailing empties

// join (Java 8+)
String joined = String.join(", ", "a", "b", "c");   // "a, b, c"
String joined2 = String.join("-", List.of("x","y")); // "x-y"
```

### Search & Match

```java
String s = "Hello World";

s.matches("Hello.*");        // true — full string must match regex
s.contains("llo");           // true
s.startsWith("He", 0);      // true — overload with offset
```

### Conversion

```java
// to char array and back
char[] chars = "Hello".toCharArray();
String back = new String(chars);
String back2 = String.valueOf(chars);

// from primitives
String.valueOf(42);          // "42"
String.valueOf(3.14);        // "3.14"
String.valueOf(true);        // "true"
String.valueOf('A');         // "A"
Integer.toString(42);        // "42"

// to number
Integer.parseInt("42");      // 42
Double.parseDouble("3.14");  // 3.14

// formatted output (Java 15+)
String s = "Hello %s, you are %d years old".formatted("Vidhan", 25);
// or
String s2 = String.format("%.2f", 3.14159);  // "3.14"
```

### Repeat, Lines, Chars (Java 11+)

```java
"ab".repeat(3);              // "ababab"

"line1\nline2\nline3"
    .lines()                 // Stream<String>
    .forEach(System.out::println);

"Hello"
    .chars()                 // IntStream of char code points
    .forEach(c -> System.out.print((char) c));

"Hello"
    .codePoints()            // IntStream of Unicode code points
    .count();                // 5
```

---

## 5. StringBuilder vs StringBuffer vs String

| Feature              | `String`         | `StringBuilder`   | `StringBuffer`    |
|----------------------|------------------|-------------------|-------------------|
| Mutability           | Immutable        | Mutable           | Mutable           |
| Thread Safety        | Safe (immutable) | **Not** safe      | Safe (synchronized)|
| Performance          | Slow (concat)    | **Fastest**       | Slower (locks)    |
| Use Case             | Constants, keys  | Single-threaded   | Multi-threaded    |
| Since                | Java 1.0         | Java 5            | Java 1.0          |

### StringBuilder In Depth

```java
StringBuilder sb = new StringBuilder();          // default capacity: 16 chars
StringBuilder sb2 = new StringBuilder(64);       // initial capacity
StringBuilder sb3 = new StringBuilder("Hello");  // initial content (len+16 capacity)

sb.append("Hello");
sb.append(' ').append("World");      // chaining
sb.insert(5, ",");                   // "Hello, World"
sb.delete(5, 6);                     // "Hello World"
sb.deleteCharAt(5);                  // "HelloWorld"
sb.replace(0, 5, "Hi");             // "HiWorld"
sb.reverse();                        // "dlroWiH"
sb.setCharAt(0, 'X');               // "XlroWiH"

sb.length();                         // current length
sb.capacity();                       // internal buffer size
sb.ensureCapacity(100);             // grow if needed
sb.trimToSize();                     // shrink buffer to length

String result = sb.toString();
```

### Capacity & Resizing

```java
// Internal growth formula: newCapacity = (oldCapacity * 2) + 2
// Start: 16
// After first overflow: 34
// After second: 70
// etc.

// Pre-sizing is a best practice when you know the output size
int expectedLength = 1_000;
StringBuilder sb = new StringBuilder(expectedLength);
```

### When the Compiler Uses StringBuilder

```java
// You write:
String s = "Hello" + " " + name + "!";

// Compiler (pre-Java 9) transforms to:
String s = new StringBuilder()
    .append("Hello")
    .append(" ")
    .append(name)
    .append("!")
    .toString();

// Java 9+ uses invokedynamic + StringConcatFactory — even more efficient
```

**But this optimization does NOT apply inside loops!**

```java
// BAD — creates a new StringBuilder + toString on EVERY iteration
String result = "";
for (String item : list) {
    result += item;   // O(n²) total
}

// GOOD — single StringBuilder, O(n) total
StringBuilder sb = new StringBuilder();
for (String item : list) {
    sb.append(item);
}
String result = sb.toString();
```

---

## 6. String Comparison Deep Dive

### `==` vs `.equals()` vs `.compareTo()`

```java
String a = "Hello";
String b = "Hello";
String c = new String("Hello");

// == compares REFERENCES (memory addresses)
a == b   // true  — both point to pool
a == c   // false — c is on heap

// equals() compares CONTENTS
a.equals(c)  // true

// equalsIgnoreCase()
"HELLO".equalsIgnoreCase("hello")  // true

// compareTo() — lexicographic order (char by char)
"apple".compareTo("banana")   // negative (a < b)
"banana".compareTo("apple")   // positive
"apple".compareTo("apple")    // 0

// Null-safe comparison (Java 7+)
Objects.equals(a, null)   // false, no NullPointerException
Objects.equals(null, null)// true
```

### Sorting Strings

```java
List<String> names = Arrays.asList("Charlie", "alice", "Bob");

// Natural order (case-sensitive: uppercase before lowercase)
Collections.sort(names);   // [Bob, Charlie, alice]

// Case-insensitive
names.sort(String.CASE_INSENSITIVE_ORDER);   // [alice, Bob, Charlie]

// By length, then alphabetical
names.sort(Comparator.comparingInt(String::length).thenComparing(Comparator.naturalOrder()));
```

### hashCode() Contract

```java
// Two equal strings MUST have the same hashCode
String s1 = "Hello";
String s2 = new String("Hello");
s1.equals(s2);                      // true
s1.hashCode() == s2.hashCode();     // true (guaranteed)

// hashCode formula (polynomial rolling hash):
// s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
// 31 is chosen: prime, odd, compiler can optimize (31*x = 32*x - x = x<<5 - x)
```

---

## 7. Encoding, Chars, and Unicode

### char vs Code Point

```java
// char is a 16-bit UTF-16 unit, NOT a Unicode code point
// Characters in BMP (Basic Multilingual Plane, U+0000–U+FFFF) → 1 char
// Supplementary characters (U+10000+) → 2 chars (surrogate pair)

String emoji = "😀";
emoji.length();               // 2 — two UTF-16 chars (surrogate pair)
emoji.codePointCount(0, 2);   // 1 — one Unicode code point

// Iterating properly over code points
"Hello 😀".codePoints()
    .forEach(cp -> System.out.print(Character.toChars(cp)));
```

### Charset Encoding

```java
import java.nio.charset.StandardCharsets;

String s = "Hello";

// String → bytes
byte[] utf8Bytes = s.getBytes(StandardCharsets.UTF_8);
byte[] utf16Bytes = s.getBytes(StandardCharsets.UTF_16);

// bytes → String
String decoded = new String(utf8Bytes, StandardCharsets.UTF_8);

// Always specify charset explicitly — default platform charset can vary
```

---

## 8. Regular Expressions with String

### Quick Reference

```java
String text = "Order #12345 placed on 2024-01-15";

// matches() — entire string must match
"12345".matches("\\d+");             // true
"abc123".matches("\\d+");           // false — not entirely digits

// replaceAll() with regex
text.replaceAll("\\d+", "NUM");     // "Order #NUM placed on NUM-NUM-NUM"

// split() with regex
"one  two   three".split("\\s+");   // ["one", "two", "three"]

// Compiled Pattern (reusable — MUCH faster in loops)
import java.util.regex.*;

Pattern p = Pattern.compile("(\\d{4})-(\\d{2})-(\\d{2})");
Matcher m = p.matcher(text);

if (m.find()) {
    System.out.println(m.group(0));  // "2024-01-15" — full match
    System.out.println(m.group(1));  // "2024" — group 1
    System.out.println(m.group(2));  // "01"   — group 2
    System.out.println(m.group(3));  // "15"   — group 3
}

// Find all matches
while (m.find()) {
    System.out.println(m.group());
}

// Named groups (Java 7+)
Pattern named = Pattern.compile("(?<year>\\d{4})-(?<month>\\d{2})-(?<day>\\d{2})");
Matcher nm = named.matcher("2024-01-15");
if (nm.matches()) {
    nm.group("year");   // "2024"
}
```

### Common Regex Patterns

```java
// Email validation
"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}"

// Phone number (Indian)
"[6-9]\\d{9}"

// UUID
"[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}"

// Alphanumeric only
"^[a-zA-Z0-9]+$"

// Non-greedy matching
"<.+?>"   // matches smallest possible tag
```

---

## 9. Java 8–21 String Enhancements

### Java 8

```java
// String.join()
String.join(", ", "a", "b", "c");  // "a, b, c"

// StringJoiner
StringJoiner sj = new StringJoiner(", ", "[", "]");
sj.add("a"); sj.add("b"); sj.add("c");
sj.toString();  // "[a, b, c]"

// Collectors.joining()
List.of("a","b","c").stream()
    .collect(Collectors.joining(", ", "[", "]"));  // "[a, b, c]"
```

### Java 11

```java
"  ".isBlank();               // true — whitespace-only check
"Hello\nWorld".lines();       // Stream<String>
"  hi  ".strip();             // "hi" — Unicode-aware trim
"ab".repeat(3);               // "ababab"
```

### Java 12

```java
// String.indent() — adds/removes leading whitespace
"Hello\nWorld".indent(4);    // "    Hello\n    World\n"

// String.transform() — apply a function
"hello".transform(s -> s.toUpperCase() + "!");  // "HELLO!"
```

### Java 13–14 — Text Blocks (Preview → GA in Java 15)

```java
// Text blocks — multi-line string literals
String json = """
        {
            "name": "Vidhan",
            "role": "Engineer"
        }
        """;

// Rules:
// - Indentation is stripped (up to the closing """)
// - Trailing whitespace is stripped per line
// - \n is automatic at each line end
// - Use \  at end of line to suppress newline
// - Use \s to preserve trailing space

String html = """
        <html>
            <body>Hello</body>
        </html>
        """;

// Formatted text block (Java 15+)
String query = """
        SELECT *
        FROM users
        WHERE name = '%s'
        """.formatted("Vidhan");
```

### Java 15

```java
// String.formatted() — instance method equivalent of String.format()
"Hello %s!".formatted("Vidhan");   // "Hello Vidhan!"
```

### Java 21 — String Templates (Preview, JEP 430)

```java
// NOTE: Still in preview; syntax may change
// StringTemplate.STR processor
String name = "Vidhan";
String msg = STR."Hello, \{name}!";   // "Hello, Vidhan!"

// Multi-line
String result = STR."""
    Name: \{name}
    Age:  \{25}
    """;
```

---

## 10. Performance Patterns & Pitfalls

### Pitfall 1: Concatenation in Loop

```java
// O(n²) — DON'T
String s = "";
for (int i = 0; i < 10_000; i++) {
    s += i;   // creates 10,000 String objects
}

// O(n) — DO
StringBuilder sb = new StringBuilder(50_000);
for (int i = 0; i < 10_000; i++) {
    sb.append(i);
}
String s = sb.toString();
```

### Pitfall 2: `intern()` Abuse

```java
// intern() is useful when storing millions of repeated strings
// e.g., status codes, city names, country codes
String status = response.getStatus().intern();  // deduplicates

// BUT: intern() itself has overhead (hashing + pool lookup)
// Don't use it on unique strings — you just pollute the pool
```

### Pitfall 3: Regex Recompilation

```java
// BAD — Pattern compiled on every call
boolean validate(String s) {
    return s.matches("\\d{10}");   // compiles pattern every time
}

// GOOD — compile once
private static final Pattern PHONE = Pattern.compile("\\d{10}");

boolean validate(String s) {
    return PHONE.matcher(s).matches();
}
```

### Pitfall 4: `substring()` Memory Leak (Pre-Java 7u6)

In very old JVMs, `substring()` shared the backing `char[]` with the original string, causing memory leaks. This was fixed in Java 7u6 — `substring()` now always creates a new copy.

```java
// Modern Java: this is safe, no leak
String huge = readHugeFile();
String small = huge.substring(0, 10);
huge = null;   // GC can collect the huge backing array
```

### Best Practices Summary

```java
// 1. Use StringBuilder for dynamic construction
// 2. Pre-size StringBuilder when length is predictable
// 3. Use String.join() / Collectors.joining() for simple list joins
// 4. Always use equals() not == for content comparison
// 5. Compile regex patterns as static final fields
// 6. Specify charset explicitly in getBytes() / new String(bytes)
// 7. Use text blocks for multi-line strings (Java 15+)
// 8. Prefer isEmpty() over length() == 0
// 9. Use isBlank() over trim().isEmpty() (Java 11+)
// 10. Avoid String.format() in tight loops — use StringBuilder or text blocks
```

---

## 11. Strings in Collections & Hashing

### String as HashMap Key

String is the ideal `HashMap` key because:
- Immutable → hash never changes
- `hashCode()` is cached after first call
- Well-defined, consistent `equals()` + `hashCode()` contract

```java
Map<String, Integer> wordCount = new HashMap<>();
wordCount.put("hello", 1);
wordCount.get("hello");   // works perfectly

// hashCode is computed once and cached in the 'hash' field
String key = "order-12345";
key.hashCode();   // computed, cached
key.hashCode();   // returns cached value — no recomputation
```

### String Interning & Memory Optimization

```java
// Scenario: reading 10 million log lines, each has a status field
// "SUCCESS", "FAILURE", "PENDING" — only 3 unique values

// Without intern: 10M String objects in heap
List<String> statuses = lines.stream()
    .map(line -> line.split(",")[2])
    .collect(Collectors.toList());

// With intern: only 3 String objects in pool, 10M references point to them
List<String> statuses = lines.stream()
    .map(line -> line.split(",")[2].intern())
    .collect(Collectors.toList());
```

---

## 12. Common Coding Problems & Solutions

### Reverse a String

```java
// Using StringBuilder
String reversed = new StringBuilder("Hello").reverse().toString();

// Manual (for interviews)
char[] chars = "Hello".toCharArray();
int l = 0, r = chars.length - 1;
while (l < r) {
    char tmp = chars[l];
    chars[l++] = chars[r];
    chars[r--] = tmp;
}
String result = new String(chars);
```

### Check Palindrome

```java
boolean isPalindrome(String s) {
    s = s.toLowerCase().replaceAll("[^a-z0-9]", "");
    int l = 0, r = s.length() - 1;
    while (l < r) {
        if (s.charAt(l++) != s.charAt(r--)) return false;
    }
    return true;
}
```

### Check Anagram

```java
boolean isAnagram(String a, String b) {
    if (a.length() != b.length()) return false;
    int[] freq = new int[26];
    for (char c : a.toCharArray()) freq[c - 'a']++;
    for (char c : b.toCharArray()) freq[c - 'a']--;
    for (int f : freq) if (f != 0) return false;
    return true;
}
```

### Longest Palindromic Substring (Expand Around Center)

```java
String longestPalindrome(String s) {
    if (s.isEmpty()) return "";
    int start = 0, end = 0;
    for (int i = 0; i < s.length(); i++) {
        int odd  = expand(s, i, i);
        int even = expand(s, i, i + 1);
        int len  = Math.max(odd, even);
        if (len > end - start + 1) {
            start = i - (len - 1) / 2;
            end   = i + len / 2;
        }
    }
    return s.substring(start, end + 1);
}

int expand(String s, int l, int r) {
    while (l >= 0 && r < s.length() && s.charAt(l) == s.charAt(r)) {
        l--; r++;
    }
    return r - l - 1;
}
```

### First Non-Repeating Character

```java
char firstNonRepeating(String s) {
    int[] freq = new int[128];
    for (char c : s.toCharArray()) freq[c]++;
    for (char c : s.toCharArray()) if (freq[c] == 1) return c;
    return '\0';
}
```

### Count Words

```java
long wordCount(String s) {
    return Arrays.stream(s.trim().split("\\s+")).count();
}
```

### Roman to Integer

```java
int romanToInt(String s) {
    Map<Character, Integer> map = Map.of(
        'I', 1, 'V', 5, 'X', 10, 'L', 50,
        'C', 100, 'D', 500, 'M', 1000
    );
    int result = 0;
    for (int i = 0; i < s.length(); i++) {
        int curr = map.get(s.charAt(i));
        int next = (i + 1 < s.length()) ? map.get(s.charAt(i + 1)) : 0;
        result += (curr < next) ? -curr : curr;
    }
    return result;
}
```

### Group Anagrams

```java
List<List<String>> groupAnagrams(String[] strs) {
    Map<String, List<String>> map = new HashMap<>();
    for (String s : strs) {
        char[] chars = s.toCharArray();
        Arrays.sort(chars);
        String key = new String(chars);
        map.computeIfAbsent(key, k -> new ArrayList<>()).add(s);
    }
    return new ArrayList<>(map.values());
}
```

### Longest Common Prefix

```java
String longestCommonPrefix(String[] strs) {
    if (strs == null || strs.length == 0) return "";
    String prefix = strs[0];
    for (int i = 1; i < strs.length; i++) {
        while (!strs[i].startsWith(prefix)) {
            prefix = prefix.substring(0, prefix.length() - 1);
            if (prefix.isEmpty()) return "";
        }
    }
    return prefix;
}
```

### String Compression (RLE)

```java
String compress(String s) {
    if (s.isEmpty()) return s;
    StringBuilder sb = new StringBuilder();
    int count = 1;
    for (int i = 1; i <= s.length(); i++) {
        if (i < s.length() && s.charAt(i) == s.charAt(i - 1)) {
            count++;
        } else {
            sb.append(s.charAt(i - 1));
            if (count > 1) sb.append(count);
            count = 1;
        }
    }
    return sb.length() < s.length() ? sb.toString() : s;
}
// "aabcccdd" → "a2bc3d2"
```

---

## 13. Interview Questions & Answers

### Q1: Why is String immutable in Java?

**A:** String immutability serves multiple purposes:

- **String Pool:** Pool-based sharing is only safe if objects can't be changed. Multiple references to the same pool object is fine because nobody can mutate it.
- **Thread Safety:** Immutable objects are inherently safe across threads — no synchronization needed.
- **Stable Hash Key:** `HashMap`, `HashSet` depend on consistent `hashCode()`. Because strings can't change, their hash is computed once and cached.
- **Security:** JVM class loading, network APIs, file I/O, and SecurityManager all accept `String`. A mutable String could be changed after a security check — immutability prevents this attack vector.

---

### Q2: What is the String Pool and where does it live?

**A:** The String Pool (also called the String Intern Pool) is a special memory region that stores unique string literals to avoid redundant object creation. When you write `String s = "Hello"`, the JVM checks if `"Hello"` already exists in the pool. If yes, it returns the existing reference; if no, it creates a new one.

- **Pre Java 7:** lived in `PermGen` (fixed-size, could cause `OutOfMemoryError`)
- **Java 7+:** moved to the main **Heap**, allowing it to be garbage-collected and grow dynamically

You can manually add a String to the pool using `intern()`.

---

### Q3: What is the difference between `==` and `.equals()` for Strings?

**A:**
- `==` compares **references** (memory addresses). Two `String` objects with the same content will be `==` only if they are the same object (e.g., from the pool).
- `.equals()` compares **content** (character by character).

```java
String a = "Hello";
String b = new String("Hello");
a == b        // false — different objects
a.equals(b)   // true  — same content
```

**Always use `.equals()` for content comparison.** Use `Objects.equals(a, b)` for null-safe comparison.

---

### Q4: `String` vs `StringBuilder` vs `StringBuffer` — when to use which?

**A:**
- **`String`:** When the value won't change — constants, map keys, method parameters.
- **`StringBuilder`:** Mutable, non-thread-safe string building. Use in **single-threaded** contexts (by far the most common case). Fastest.
- **`StringBuffer`:** Mutable, thread-safe (all methods `synchronized`). Use only when **multiple threads** share the same buffer — which is rare; most concurrent designs avoid shared mutable state entirely.

In practice: `StringBuilder` for 99% of cases, `String` for constants, `StringBuffer` almost never (design out shared mutable state instead).

---

### Q5: How does Java's `String.hashCode()` work?

**A:** Java uses a **polynomial rolling hash**:

```
hash = s[0]*31^(n-1) + s[1]*31^(n-2) + ... + s[n-1]
```

31 is chosen because it's a prime and odd, and `31 * x` can be optimized by the compiler to `(x << 5) - x`.

The hash is **lazily computed** (on first call) and **cached** in the private `hash` field. Subsequent calls return the cached value in O(1). This is thread-safe because: if two threads race on the first computation, both compute the same value, so the last write wins without corruption.

---

### Q6: What happens internally when you do `String s = "a" + "b" + variable`?

**A:** The compiler transforms string concatenation:
- **Compile-time constants** (`"a" + "b"`) are folded to `"ab"` at compile time and stored directly in the pool.
- **Runtime concatenation** involving variables: in Java 8 and below, the compiler generates `new StringBuilder().append("ab").append(variable).toString()`. In **Java 9+**, the compiler emits an `invokedynamic` instruction pointing to `StringConcatFactory`, which selects the most efficient strategy at runtime (often faster than the old StringBuilder approach).

**The key trap:** if concatenation is inside a loop, the compiler's optimization only covers the single concatenation expression — a `+=` in a loop creates a new `StringBuilder` and `toString()` on every iteration → O(n²). Always use an explicit `StringBuilder` outside the loop.

---

### Q7: What is `intern()` and when should you use it?

**A:** `intern()` checks the String Pool for an equal string. If found, returns the pool reference. If not, adds the string to the pool and returns it.

```java
String a = new String("Hello").intern();
String b = "Hello";
a == b   // true — both point to pool
```

**Use it when:** You have a large number of repeated strings (e.g., status codes, country names read from a file) and want to deduplicate them in memory. Calling `intern()` on 10 million `"SUCCESS"` strings reduces them to one object with 10 million references.

**Avoid it when:** Strings are mostly unique — you just add overhead with no benefit.

---

### Q8: What is a text block and why was it introduced?

**A:** Text blocks (GA in Java 15, JEP 378) allow multi-line string literals without escape sequences. They were introduced to improve readability when embedding SQL, JSON, HTML, or XML.

```java
String query = """
        SELECT id, name
        FROM users
        WHERE active = true
        ORDER BY name
        """;
```

Key behaviors: incidental leading whitespace is stripped (based on the closing `"""`), newlines are preserved, and trailing whitespace per line is stripped. The result is a regular `String` — there's no new type.

---

### Q9: How would you check if a String is a valid palindrome ignoring non-alphanumeric characters?

**A:**

```java
boolean isPalindrome(String s) {
    int l = 0, r = s.length() - 1;
    while (l < r) {
        while (l < r && !Character.isLetterOrDigit(s.charAt(l))) l++;
        while (l < r && !Character.isLetterOrDigit(s.charAt(r))) r--;
        if (Character.toLowerCase(s.charAt(l)) != Character.toLowerCase(s.charAt(r)))
            return false;
        l++; r--;
    }
    return true;
}
```

O(n) time, O(1) space — no new string/array created.

---

### Q10: How does Compact Strings (Java 9) improve memory?

**A:** Before Java 9, every `String` was backed by a `char[]` (2 bytes per character, UTF-16). In practice, most applications are ASCII-heavy (Latin script), so half the bytes in every string were wasted zeros.

Java 9 (JEP 254) changed the backing store to `byte[]` with a `coder` byte:
- If all characters fit in Latin-1 (code points ≤ 255) → 1 byte per character
- Otherwise → 2 bytes per character (UTF-16)

For typical Java applications, this reduces heap usage by **10–15%** and improves cache utilization. The public `String` API is completely unchanged — it's a purely internal optimization.

---

### Q11: Reverse words in a sentence without using split().

```java
String reverseWords(String s) {
    s = s.trim();
    StringBuilder result = new StringBuilder();
    int end = s.length();
    for (int i = s.length() - 1; i >= 0; i--) {
        if (s.charAt(i) == ' ') {
            if (end != i + 1) result.append(s, i + 1, end).append(' ');
            end = i;
        }
    }
    result.append(s, 0, end);
    return result.toString().trim();
}
```

---

### Q12: How would you find all permutations of a string?

```java
void permutations(String s, String prefix, List<String> result) {
    if (s.isEmpty()) {
        result.add(prefix);
        return;
    }
    for (int i = 0; i < s.length(); i++) {
        permutations(
            s.substring(0, i) + s.substring(i + 1),
            prefix + s.charAt(i),
            result
        );
    }
}
// Call: permutations("abc", "", new ArrayList<>())
// Result: ["abc","acb","bac","bca","cab","cba"]
```

---

### Q13: Explain the ShopSphere angle — where do String internals matter in production microservices?

**A:** Several high-impact areas:

1. **Kafka message keys:** Order IDs, user IDs used as partition keys are strings. Proper `hashCode()` behavior ensures even partition distribution.

2. **Redis cache keys:** String concatenation like `"product:" + productId + ":stock"` — use `StringBuilder` or formatted strings to build keys efficiently in hot paths.

3. **Elasticsearch queries:** JSON query bodies built with `String.format()` or text blocks — prefer pre-compiled templates.

4. **JWT parsing:** Token strings are parsed repeatedly. `split("\\.")` with a pre-compiled `Pattern` is faster than calling `String.split()` on each request.

5. **Log aggregation:** In the Search service, product names and category strings from thousands of events — `intern()` on repeated taxonomy values reduces GC pressure.

6. **SQL queries:** Text blocks make multi-line JPQL/native queries dramatically more readable and maintainable compared to concatenated string literals.

---

*End of Java Strings Deep Dive — covers theory, JVM internals, all major methods, Java 8–21 evolution, performance traps, 8 coding problems, and 13 interview Q&As.*
