# Token structure

Each token will have the following general structure:

	<header>
	<uuid>
	<expiration>
	<vocabulary header>
	<vocabulary entries ...>
	<payload header>
	<payload entries ...>
	<patterns ...>
	<signature>

## Header

The token header is one byte. Its bits have the following meaning, starting from the most significant bit.

Bit | Description
----|------------
0…3 | Version of the token, reserved for future extensions. Currently, each of these bits must be 0.
4…7 | Unsigned integer defining the signature algorithm to use, see below.

The signature algorithm can be one of the following:

Value | Algorithm
------|----------
1     | HS256
2     | HS384
3     | HS512

At the moment any other value is considered invalid. They are reserved for future extensions.


## UUID

The `<uuid>` part of the token will describe an unique identifier for this token.
It can be used e.g. for invalidation of tokens before they expire.
The identifier is always present and it also works as a "salt" to the signature hash algorithm.

The `<uuid>` part is 16 bytes (128 bits) long.
It should represent a valid [Universally unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier).


## Expiration

The expiration time on or after which the token **must not** be accepted for processing.
The date/time is represented as an [UNIX timestamp](http://pubs.opengroup.org/onlinepubs/9699919799/basedefs/V1_chap04.html#tag_04_16), or, "seconds since Epoch".
The expiration time is 5 bytes. (40 bits, which will avoid the [Year 2038 problem](https://en.wikipedia.org/wiki/Year_2038_problem) with 32 bits).


## Vocabularies

A vocabulary is an indexed list of commonly occuring strings that can be referenced from the string sequences in order to preserve space.
There are two kinds of vocabularies: a _bundled vocabulary_ and an _external vocabulary_.
Any string sequence in the token, in the payload or in the path patterns, may refer to a string in either type of vocabulary **with a single byte**.

Either type of vocabulary may store the maximum number of 64 string sequences.

### Bundled vocabulary

A bundled vocabulary is embedded to the token itself. It will contain string sequences that occur multiple times in either the payload or the path patterns within the token.

When packing a token, a bundled vocabulary will be automatically built to minimize the total size of the token.

The bundled vocabulary, if included in the token, will start with the `<vocabulary header>` byte, which will describe the number of words in the vocabulary, in their indexed order. The most significant bit must be 0, and the other 7 bits are interpreted as an unsigned integer.

The header is followed by the given number of string sequences (`<vocabulary entries ...>`). The first byte of each entry is interpreted as a signed integer describing the total number of string bytes in the sequence. This length byte is then followed by that number of [string bytes](#string-byte).

Note that the string bytes may contain references to other vocabulary string sequences, either in the bundled vocabulary or in the external vocabulary. However, in order to avoid infinitely recursive loops, a bundled vocabulary entry string may only refer to another bundled entry whose index is smaller than the referring entry.

### External vocabulary

An external vocabulary is a list of strings that is not included in the packaged token, but is known by the code that is packing **and** unpacking the token! Any occurence of the string in an external vocabulary will only take byte in the packed token, no matter how long the string is.

Providing a custom external vocabulary is optional. There is a _default external vocabulary_ that is automatically used if you do not provide one yourself.
The default vocabulary will contain some common words that are generally used in URL paths.
This default vocabulary will not be changed as long as the token version remains the same, so its safe to use.

The default vocabulary consists of the following string sequences:

- account
- action
- admin
- album
- api
- app
- audio
- auth
- categor
- chat
- client
- comment
- connection
- countr
- develop
- doc
- domain
- exp
- friend
- game
- group
- image
- key
- label
- language
- link
- location
- login
- mail
- membership
- message
- object
- organization
- page
- photo
- place
- post
- prod
- product
- profile
- request
- resource
- response
- room
- share
- status
- tag
- team
- token
- user
- value
- video
- visitor

It's important to note that you **must provide the same vocabulary wherever the token is packed or unpacked!**
Usually, once defined for the first time, it is not safe to change the external vocabulary, because your code may receive an old, non-expired token that was based to the old vocabulary. You should pay special attention when extending or updating an external vocabulary.


## Payload

The token may contain an optional **payload**: a number of key-value pairs of additional data to be bundled with the token.

In order to ensure compact representation, there are strict limits on what kind of data the token may contain.
The payload is intended to be used to store e.g. user or account identifiers, additional timestamps, or other simple information.

The following limitations apply:

- The *keys* are strings containing only [basic ASCII characters](http://www.asciitable.com/) with the maximum length of 127 characters.
- The *values* may be either:
	- String containing only [basic ASCII characters](http://www.asciitable.com/) with the maximum length of 127 characters.
	- [Universally unique identifier](https://en.wikipedia.org/wiki/Universally_unique_identifier) (128 bits)
	- Signed 64 bit integer (in range -9,223,372,036,854,775,808 … 9,223,372,036,854,775,807)
	- Boolean value of true or false
	- List of values of any of the types above, with the maximum size of 63 items.

**NOTE:** Payload is likely to increase the size of the token, so only include as little additional data as possible!
If you need a large or complex payload, or you need to include unsupported data types, you may want to consider using [JSON Web Tokens](https://jwt.io/) instead.

The `<payload header>` is a single byte that represents the number of key-value pairs that included in the token.
This number of key-value pairs are expected to follow this header (`<payload entries ...>`).

Each payload entry is a tuple of two payload items. The first one represents the key, and the second one the value.

For both key and value, there is byte describing the type and the length of the item. The following rules apply:

- If the _most significant bit_ is 0, then the type is *string*, and the other 7 bits will describe how many bytes will describe the string sequence. For payload keys, this must always be 0.
- If the _most significant bit_ is 1, then the type is determined by the next bit:
	- If the next bit is 0, the type is *list*, and the other 6 bits will describe the number of items in the list as an unsigned integer.
	- If the next bit is 1, then the type is determined by the remaining 6 bits:
		- 000000: The value is a boolean value of _false_
		- 000001: The value is a boolean value of _true_
		- 000010: The type is a signed 64 bit integer. The value will have the length of 8 bytes.
		- 000011: The type is an UUID. The value will have the length of 16 bytes (128 bits).

After this byte, the following bytes will describe the key/value, depending on the type.
In some cases, e.g. when the type was true/false, the value will be omitted.
However, if the type was a list, then there may be more than one value items.

If the type as a string, then the following bytes will be interpreted as [string bytes described below](#string-byte).


## Patterns

### Command bytes

The two (2) most significant bits of the byte define a type of this command byte as an integer in range 0…3.

Value | Type
------|------
0     | Start a string sequence
1     | Define a set of methods
2     | Start a nested list of items
3     | Reserved for future extensions

The meaning of the other 6 bits depends on the type.

**Command value 0** indicates that the next bytes must be interpreted as [string bytes](#string-byte).
The 6 least significant bits are described as an integer, describing the number of string bytes in the following sequence.
After that number of string bytes, the next following byte is again interpreted as a command byte.

**Command value 1** indicates that this byte describes the set of allowed methods for the current path sequence.
Each of those bits will present a HTTP method: if 1, the method is allowed, if 0, the method is disallowed.

Nth significant bit | HTTP method
--------------------|------------
2                   | GET
3                   | HEAD
4                   | POST
5                   | PUT
6                   | PATCH
7                   | DELETE

This byte will terminate the current item. The next byte will be interpreted as a command byte that either starts a new item in this or on an outer level, depending on the number of items that have been occurred on this nested level.

**Command value 2** indicates that this byte will start a _nested level of items_.
The 6 least significant bits are described as an integer that will tell how many items will follow on this nested level.
Note that nested items will themselves start nested levels, whose sub-items are not counted in this number.

Command | Description
--------|------------
0       | This byte continues the previous string. The last 6 bits of this byte define _a character_.
1       | This byte starts _a nested level of sequences_. Each of the nested sequences are prefixed with the string defined by previous bytes. The last 6 bits of this byte describe the _number of nested items_. Item can be either a string terminated with a set of methods (command 2) or it can be a string that followed by another level of nested items (command 1).
2       | This byte will terminate the previous string. The last 6 bits of this byte define a _set of HTTP methods_. The next byte will start at a _sibling_ sequence.
3       | Reserved for future extensions.


### String byte

The *most significant bit* describes the type of this string item:

- Bit 0 means the byte describes _an ASCII character_
- Bit 1 means the byte describes _a string sequence from a vocabulary_

In case of 0, the other 7 bits describe [an ASCII character code](http://www.asciitable.com/).

In case of 1, _the second most significant bit_ describes which vocabulary is used:

- Bit 0 means we are referencing the _bundled vocabulary_.
- Bit 1 means we are referencing the _external vocabulary_.

In both cases, the last 6 bits represent an integer in range 0…63 that describe the index of the string in the referenced vocabulary.
The byte will represent this string sequence.


## Signature

The rest of the token is a signature that is used to verify that the token was created by a trusted source and has not been tampered with.
The signature is created with the followin pseudocode:

    HMAC(token_body + external_vocabulary, secret)

- The `token_body` is all the byte contents of the token before the signature part.
- The `external_vocabulary` is any external vocabulary used for the token contents. The external vocabulary must be represented in the same way that an internal vocabulary would, including the header byte. This also applies to the default external vocabulary.
- The `secret` is a string key only known by the trusted source. The `HMAC` algorithm depends on the definition in the header byte of the token.
