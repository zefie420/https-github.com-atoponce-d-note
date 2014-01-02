d-note
======

Introduction
------------

d-note is a self destructing notes application you can run on your own web
server. I got the idea from a number of websites doing pretty much the same
thing:

* https://oneshar.es/
* https://privnote.com/
* https://quickforget.com/
* https://onetimesecret.com/
* https://burnnote.com/

And many more. Unfortunately, none of the above sites seem to be interested
in benefiting the community as a whole by providing their source code, even
though there seems to be a demand for it. Further, by using a third party
service, you have no guarantee that they are not sharing your data with the
NSA. By running your own server, you know who has your data and who doesn't.

The name of the project is inspired by the "H-Bomb", or hydrogen bomb. I
wanted a clever name for self destructing notes that was not in use, and
something that had a familiar ring to it. "d-note" seemed to fit for
"destructive note", and as already mentioned, inspired by the hydrogen
bomb.

Securing The Data
=================

Server Side
-----------

All notes are compressed using ZLIB and encrypted using Blowfish in ECB mode
before being written to disk. Never at any point is the note in plaintext on
disk.

Blowfish was chosen as the cipher for encrypting the data, as the key can be
any length, whereas AES requires static key lengths. This could be enforced
client-side with Javascript, but some clients have Javascript disabled.

ECB mode was chosen over CBC for encrypting the blocks, due to not wanting to
maintain an initialization vector. Even though ECB can leack information about
plaintext blocks, the data is compressed with ZLIB before encrypted. This
increases the entropy of the plaintext, and removes duplicated blocks from the
plaintext, as is the design of compression algorithms.

Although I cannot enforce this, you should serve the application over SSL. The
plaintext must be transmitted between the browser and the server, before it can
be encrypted. The data is also decrypted on the server, and transmitted to the
recipient's browser. As such, you should protect the data from being sniffed
using SSL on this application.

When the note is destroyed, it is first overwritten 3 times with random data,
then removed. There are a couple benefits from this:

* If the underlying filesystem is copy on write, then new random data is
  written to new sectorns on disk, making it difficult to know precisely where
  the encrypted file ended and where the random data started.
* If the underlying filesystem is a journaled filesystem, the journal may have
  logged entries of the data. But again, because it's encrypted then random, it
  will be very difficult to know when the encrypted data was stored and when
  it was overwritten with random data.

Client Side
-----------

There are many things I could do to greatly discourage the recipient from
copying the plaintext, but they're mostly just annoying. There's nothing to
stop the recipient from memoring the note, taking a screenshot (or many), or
copying and pasting the text. If the recipient really wants the note saved that
badly, it will happen. As such, the only thing that can be done is destroying
the note after a given interval, to prevent it from being viewed again.

Python Documentation
====================

URL Generation
--------------

URLs for self destructing notes should not be predictable in any manner.
Thus, a 22-character base64-encoded string is generated for each
submission. This will give us enough random URLs to avoid a collision with
1 in 2^128. The code should be self-documenting, however, this might
explain things a bit more clearly.

Each URI is built using a random string of data from the Python Crypto library,
built from a 128-bit, or 16-byte number. The URL is then base64 encoded, with
URL-safe characters.

We then encode the string using `base64.urlsafe_b64encode(u.bytes)[:22]`
from the base64 module. This gives us 22 characters for our URL. The valid
characters for our URLs are thus:

    ABCDEFGHIJKLMNOPQRSTUPVXYZabcdefghijklmnopqrstupvxyz0123456789-_

So, a valid URL for your self destructing notes could be:

    https://example.com/cWQI4m3fRcW8zM_Mdeg3uQ

There are some notes to consider with this URL scheme:

Its format is XXXXXXXXXXXXXXXXXXXXXXO, where 'O' is either AgQw.

Regardless, the server would need to be processing 1 billion URLs every
second for 1,000 years before we reached the probability of 1/2 for
generating a duplicate URL.

d-note does not keep track of which URLs have been generated. Thus, it is
possible, although highly improbable, that the same URL could be generated
for two different form submissions. Of course, the application will check
against any valid notes that have not yet self destructed, but will not
check for ones that have.

Thus, it is possible that a URL that has already self destructed could be
regenerated at a different time, which has not self destructed. If the
first URL is publicly accessible, that means that the second URL could be
opened by the wrong recipient accidentally. As such, these URLs should be
kept as private as possible to prevent this from happening.

JavaScript Documentation
========================

Hashcash
--------

The point of a Hashcash implementation is to prevent form spam. I'm not
sure what the benefit of spammers would be to use self-destructing notes,
but nonetheless, I'm not really interested in entertaining it.
Implementaning Hashcash as a proof-of-work system is simple enough to
deter most spammers. The breakdown is as follows:

* Server gets client IP address.
* Server generates nonce.
* IP address and nonce become the resource string.
* The resource string is embedded invisibly into the form.
* Client then mints a Hashcash token based on the resource string found.
* The client submits the form with the minted token.
* Server verifies if the token is valid.
    * If valid, the form submits.
    * If not valid, the user is notified submission failed.

A resource string with IP address "65.100.223.163", and nonce "VILymxxv"
generated by the server could look like this:

    65100223163VILymxxv

A minted Hashcash token generated by the client would then need to look
something like this:

    1:20:120715:65100223163vilymxxv::lVb6gfTxYb1Ir5SW:COBt

This is valid, because the SHA1 hash of the above token is:

    00000bed07757ed13f1f2cebc67b616c75812b41

which starts with 20-bits of leading zeros. The work is forced on the
client, which inserts the token into the form. Even on modern hardware,
this should be a strenuous task on the client CPU, and could take up to a
second or two to create a valid token string. However, the server can
verify the token quickly (in nanoseconds).

The minting of the token should be done in the background while the user is
typing the note in the form. Thus, when the submit button is pressed, no
additional waiting is needed.

More info can be found at http://hashcash.org.
