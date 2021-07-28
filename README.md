# Runes - Because You Didn't Actually Want a MAC

https://research.google/pubs/pub41892/ is a paper called "Macaroons:
Cookies with Contextual Caveats for Decentralized Authorization in the
Cloud".  It has one good idea, some extended ideas nobody implements,
and lots and lots of words.

The idea: a server issues a cookie to Alice.  She can derive cookies
with extra restrictions and hand them to Bob and Carol to send back to
the server, and they can't remove the restrictions.

But they did it using a Message Authetication Code (MAC, get it?),
which is actually counter-productive, since it's simpler and better to
use Length Extension to achieve the same results.  I call that a Rune;
this version really only handles strings, but you can get creative.

## Rune Language

A *rune* is a series of restrictions; you have to pass all of them (so
appending a new one always makes the rune less powerful).  Each
restriction is one or more alternatives ("cmd=foo OR cmd=bar"), any
one of which can pass.

The form of each alternative is a simple string:

    ALTERNATIVE := FIELDNAME CONDITION VALUE

`FIELDNAME` and `VALUE` contain only UTF-8 characters, exclusive of
! " # $ % & ' ( ) * +, - . / : ;  ? @ [ \ ] ^ _ ` { | } ~ (C's ispunct()).


`CONDITION` is one of the following values:
* `!`: Pass if field is missing (value ignored)
* `=`: Pass if exists and exactly equals
* `^`: Pass if exists and begins with
* `$`: Pass if exists and ends with
* `~`: Pass if exists and contains
* `<`: Pass if exists, is a valid decimal (may be signed), and numerically less than
* `>`: Pass if exists, is a valid decimal (may be signed), and numerically greater than
* `}`: Pass if exists and lexicograpically greater than (or longer)
* `{`: Pass if exists and lexicograpically less than (or shorter)
* `#`: Always pass: no condition, this is a comment.

Grouping using `(` and `)` may be added in future.

A restriction is a group of alternatives separated by `|`; restrictions
are separated by `&`.
e.g.
    cmd=foo|cmd=bar&subcmd!|subcmd{get

The first requires `cmd` be presebt, and to be `foo` or `bar`.  The second
requires that `subcmd` is not present, or is lexicographically less than `get`.
Both must be true for authorization to succeed.


## Rune Authorization

Following the Rune comes a SHA-256 authentication code.  This is
generated as SHA-256 of the following bytestream:

1. The secret (less than 56 bytes, known only to the server which issued it).
2. For every restriction:
   1. Pad the stream as per SHA-256 (i.e. append 0x80, then zeroes, then
      the big-endian 64-bit bitcount so far, such that it's a multiple of 64
      bytes).
   2. Append the restriction.

By using the same padding scheme as SHA-256 usually uses to end the
data, we have the property that we can initialize the SHA-256 function
with the result from any prior restriction, and continue.

The server can validate the rune authorization by repeating this
procedure and checking the result.

## Rune Encoding

Runes are encoded as base64, starting with the 256-bit SHA256
authentication code, the followed by one or more restrictions
separated by `&`.

Not because base64 is good, but because it's familiar to Web people;
we use RFC 3548 with `+` and `/` replaced by `-` and `_` to make
it urlsafe.

## API Example

Here's the server, making you a rune! (spoiler: it's
"-YpZTBZ4Tb5SsUz3XIukxBxR619iEthm9oNJnC0LxZM=")

```
import runes
import secrets

# Secret determined by fair dice roll.
secret = bytes([5] * 16)

# Make an unrestricted rune.
rune = runes.MasterRune(secret)

# We could add our own restrictions here, if we wanted.
print("Your rune is {}".format(rune.to_str()))
```

Here's the server, checking a rune:

```
import runes
import time
import sys

secret = bytes([5] * 16)
master_rune = runes.MasterRune(secret)

# In real life, this would come from the web data.
runestring = sys.argv[1]

# You'd catch exceptions here, usually.
rune = runes.from_str(runestring)

# Make sure auth is correct, first.
if not master_rune.is_rune_authorized(rune):
    print("Rune is not authorized, go away!")
    return

# Now, lets see if it meets our values.  I assume we
# have values time (UNIX, seconds since 1970), command
# and optional id.
pass, whyfail = rune.are_restrictions_met({'time': int(time.time),
                                           'command': 'somecommand',
                                           'id': 'DEADBEEF'})
if not pass:
    print("Rune restrictions failed: {}".format(whyfail))
    return

print("Yes, you passed!")
```


Here's the client Alice.  She gets the rune and gives Bob a variant
that can only be used for 1 minute:

```
import runes
import time

# In real life, this would come from the web data.
runestring = sys.argv[1]

# You'd catch exceptions here, usually.
rune = runes.from_str(runestring)

# You can construct a Restriction class from a sequence of Alternative
# but it's easier to use decode() to translate a string
rune.add_restriction(rune.Restriction.decode("time < {}".format((int)time.time() + 60))

print("Your restricted rune is {}".format(rune.to_str()))
```







