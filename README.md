# cppcodec

Header-only C++11 library to encode/decode base64, (Crockford) base32 and hex (a.k.a. base16).
MIT licensed with consistent, flexible API. Supports raw pointers, `std::string` and
(templated) character vectors without unnecessary allocations.



# Usage

1. Import cppcodec into your project (copy, git submodule, etc.)
2. Add the cppcodec root directory to your build system's list of include directories
3. Include headers and start using the API.

Since cppcodec is a header-only library, no extra build step is needed.
Alternatively, you can install the headers and build extra tools/tests with CMake.



# Variants

A number of codec variants exist for base64 and base32, defining different alphabets
or specifying the use of padding and line breaks in different ways. cppcodec is designed
to let you make a conscious choice about which one you're using, but assumes you will
mostly stick to a single one.

cppcodec's approach is to implement encoding/decoding algorithms in different namespaces
(e.g. `cppcodec::base64_utf7`) and in addition to the natural headers, also offer
convenience headers to define a shorthand alias (e.g. `base64`) for one of the variants.

Here is an expected standard use of cppcodec:

```C++
#include <cppcodec/base64_default_utf7.hpp>
#include <cppcodec/base32_default_crockstr.hpp>
#include <iostream>

void main() {
   std::vector<char> decoded = base64::decode("YW55IGNhcm5hbCBwbGVhc3VyZS4");
   std::cout << base32::encode(decoded); // "any carnal pleasure."
}
```

If possible, avoid including "default" headers in other header files.

Non-aliasing headers omit the "default" part, e.g. `<cppcodec/base64_utf7.hpp>`
or `<cppcodec/hex_lowercase.hpp>`. Currently supported variants are:

* `base64_utf7` uses the PEM/MIME/UTF-7 alphabet, that is (in order) A-Z, a-z, 0-9 plus
  characters '+' and '/'. This is what's usually considered "standard base64",
  but without the padding ('=') and line breaks of PEM and MIME variants.
* `base32_crockstr` uses the [Crockford variant](http://www.crockford.com/wrmg/base32.html).
  It's less widely used than the RFC 4648 alphabet, but is the most sensible of
  of base32 alphabets and also accepts lower-case letters when decoding.
  Crockford base32 uses no '=' padding. Checksums are not implemented. Note that
  the specification is ambiguous about whether to pad bit quintets to the left or
  to the right, i.e. whether the codec is a place-based single number encoding
  system or a concatenative iterative stream encoder. This codec variant picks
  the streaming interpretation and thus zero-pads on the right. (See
  http://merrigrove.blogspot.ca/2014/04/what-heck-is-base64-encoding-really.html
  for a detailed discussion of the issue.)
* `hex_lowercase` outputs lower-case letters and accepts upper-case as well.



# API

All codecs expose the same API. In the below documentation, replace `<codec>` with a
default alias such as `base64`, `base32` or `hex`, or with the full namespace such as
`cppcodec::base64_utf7` or `cppcodec::base32_crockstr`.

For templated parameters `T` and `Result`, you can use e.g. `std::vector<char>`,
`std::string` or anything that supports:
* `.data()` and `.size()` for `T` (read-only) template parameters,
* for `Result` template parameters, also `.reserve(size_t)`, `.resize(size_t)`
  and `.push_back([unsigned] char)`.

It's possible to support types lacking these functions, consult the code directly if you need this.


## Encoding

```C++
// Convenient version, returns an std::string.
std::string <codec>::encode(const [unsigned] char* binary, size_t binary_size);
std::string <codec>::encode(const T& binary);

// Convenient version with templated result type.
Result <codec>::encode<Result>(const [unsigned] char* binary, size_t binary_size);
Result <codec>::encode<Result>(const T& binary);

// Reused result container version. Resizes encoded_result before writing to it.
void <codec>::encode(Result& encoded_result, const [unsigned] char* binary, size_t binary_size);
void <codec>::encode(Result& encoded_result, const T& binary);
```

Encode binary data into an encoded (base64/base32/hex) string.
Won't throw by itself, but the string type might throw on `.resize()`.

```C++
size_t <codec>::encode(char* encoded_result, size_t encoded_buffer_size, const [unsigned] char* binary, size_t binary_size) noexcept;
size_t <codec>::encode(char* encoded_result, size_t encoded_buffer_size, const T& binary) noexcept;
```

Encode binary data into pre-allocated memory with a buffer size of
`<codec>::encoded_size(binary_size)` or larger.

Returns the byte size of the encoded string, which is equal to `<codec>::encoded_size(binary_size)`.

If `encoded_buffer_size` is larger than required, a single null termination character (`'\0'`)
is written after the last encoded character. The `encoded_size()` function ensures that the required
buffer size is large enough to hold the padding required for the respective codec variant.

Calls abort() if `encoded_buffer_size` is insufficient. (That way, the function can remain `noexcept`
rather than throwing on an entirely avoidable error condition.)

```C++
size_t <codec>::encoded_size(size_t binary_size) noexcept;
```

Calculate the (exact) length of the encoded string based on binary size,
excluding null termination but including padding (if specified by the codec variant).


## Decoding

```C++
// Convenient version, returns an std::vector<char>.
std::vector<char> <codec>::decode(const char* encoded, size_t encoded_size);
std::vector<char> <codec>::decode(const T& encoded);

// Convenient version with templated result type.
Result <codec>::decode<Result>(const char* encoded, size_t encoded_size);
Result <codec>::decode<Result>(const T& encoded);

// Reused result container version. Resizes binary_result before writing to it.
void <codec>::decode(Result& binary_result, const char* encoded, size_t encoded_size);
void <codec>::decode(Result& binary_result, const T& encoded);
```

Decode an encoded (base64/base32/hex) string into a binary buffer.

```C++
size_t <codec>::decode([unsigned] char* binary_result, size_t binary_buffer_size, const char* encoded, size_t encoded_size);
size_t <codec>::decode([unsigned] char* binary_result, size_t binary_buffer_size, const T& encoded);
```

Decode an encoded string into pre-allocated memory with a buffer size of
`<codec>::decoded_max_size(encoded_size)` or larger.

Returns the byte size of the decoded binary data, which is less or equal to
`<codec>::decoded_max_size(encoded_size)`.

Calls abort() if `binary_buffer_size` is insufficient (for consistency with encode()).
Throws a cppcodec::parse_error exception (inheriting from std::domain_error)
if the input data does not conform to the codec variant specification.

```C++
size_t <codec>::decoded_max_size(size_t encoded_size) noexcept;
```

Calculate the maximum size of the decoded binary buffer based on the encoded string length.

If the codec variant does not allow padding or line breaks, the maximum decoded size will be the exact decoded size.

If the codec variant allows padding or line breaks, the actual decoded size might be smaller.
If you're using the pre-allocated memory result call, make sure to take its return value
(the actual decoded size) into account.