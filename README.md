`ewp-rsa-aes128gcm` Response Encryption
=======================================

This document describes how to accomplish confidentiality of EWP HTTP responses
with the use of `ewp-rsa-aes128gcm` encryption.

* [What is the status of this document?][statuses]
* [See the index of all other EWP Specifications][develhub]


Introduction
------------

This method of response encryption can be used, when - for any reason - using
plain TLS is "not enough". It makes use of `ewp-rsa-aes128gcm` encryption, as
specified [here][encr-spec].

Note, that only the body of the response is encrypted. Other response
parameters (such as HTTP response status and response headers) remain
unencrypted.


Implementing a server
---------------------

### Process the `Accept-Encoding` header

Check the contents of the request's `Accept-Encoding` header, as described in
[this RFC 7231 chapter][accept-encoding-rfc]. If one of the codings listed
there is `ewp-rsa-aes128gcm` (with `qvalue` greater than zero), then the client
indicates that it wants you to encrypt your response, and you MUST do so (you
cannot ignore the `ewp-rsa-aes128gcm` entry, even if other codings are present
in `Accept-Encoding` header).

If `ewp-rsa-aes128gcm` is not present among the codings, then the client
doesn't want you to use this encryption method. You should proceed and verify
if the client doesn't want you to use some other encryption method (e.g. fall
back to the regular [TLS response encryption][resencr-tls]).

If `ewp-rsa-aes128gcm` is the only encryption method you support (i.e. you
don't allow plain TLS encryption), but the client doesn't include
`ewp-rsa-aes128gcm` among its `Accept-Encoding` codings, then you MUST respond
with a HTTP 4xx error response (preferably HTTP 406), and include a proper
`<developer-message>`.


### Determine which key to use

Check for an `Accept-Response-Encryption-Key` header in the request. If it is
present, and it contains a Base64-encoded RSA public key, then this is the key
to use. If it is present, but it doesn't contain a proper Base64-encoded RSA
public key, then you SHOULD respond with HTTP 400 error.

If it is not present, but the client used [HTTP Signature Client
Authentication][cliauth-httpsig], then you MUST use the same key as the one
identified by `keyId` parameter of the `Authorization` request header.

If it is not present, and the client used some other client authentication
method involving an RSA key-pair, then you MAY use a public key from the same
key-pair for encryption. You are NOT REQUIRED to do so however - you are only
required to support this key-detection feature for HTTP Signature Client
Authentication, and only in case when you actually support HTTP Signature
Client Authentication at this endpoint).

If you cannot determine the encryption key, then you MUST respond with HTTP
4xx error response (preferably HTTP 406), and include a proper
`<developer-message>`.


### Include `Content-Encoding` header

You MUST include a proper `Content-Encoding` header, as described in [this RFC
7231 chapter][content-encoding-rfc], with `ewp-rsa-aes128gcm` listed as one of
the applied codings.

It is RECOMMENDED to additionally support the `gzip` encoding. If the client
includes `gzip` it its `Accept-Encoding` request header, then you will be able
to significantly reduce the size of your response by first compressing it, and
then encrypting the compressed body.

Note, that all other headers remain "as they were". For example, if you're
encrypting an XML document, then the encrypted response will still have the
same `text/xml` or `application/xml` `Content-Type`.


### Encrypt the response

When applying your `ewp-rsa-aes128gcm` coding, encrypt the payload as described
in the [`ewp-rsa-aes128gcm` Encryption Specs][encr-spec].


Implementing a client
---------------------

### Include `Accept-Encoding` header

The client notifies the server that it wants the server to encrypt the
response. It does so by including `ewp-rsa-aes128gcm` encoding
(case-insensitive) among the values listed in the request's `Accept-Encoding`
header.

You MAY indicate additional encodings you support. In particular, it is
RECOMMENDED to support the `gzip` encoding. You MUST follow the [RFC 7231
guidelines][accept-encoding-rfc] when constructing your `Accept-Encoding`
header. For example, if you send your request with the following header:

```http
Accept-Encoding: deflate, ewp-rsa-aes128gcm, gzip
```

Then the server MAY, for example, respond with `Content-Encoding: gzip,
ewp-rsa-aes128gcm` which indicates that the response has been first gzipped,
and *then* encrypted (so you will need to first decrypt, and then decompress
such response).


### Include `Accept-Response-Encryption-Key` header (optional)

This header is:

 * OPTIONAL, if you have used a [HTTP Signature Client
   Authentication][cliauth-httpsig]. In this case, the server is REQUIRED to
   encrypt its response for the same RSA key with which your request has been
   signed.

 * RECOMMENDED, if you have used some other client authentication method
   involving RSA key-pair. In this case, the server is NOT REQUIRED to be able
   to automatically extract your key-pair from the client authentication, so
   it's much safer to provide it explicitly.

 * REQUIRED, if your request is anonymous (unauthenticated), or your client
   authentication method does not involve any RSA key-pair. In this case, the
   server cannot know which public key to use for encrypting the response, and
   you need to provide it explicitly.

If provided, then it contains a Base64-encoded RSA public key (*not* the
fingerprint of the key, but the actual key). It MAY be used by authenticated
clients to override the default choice of encryption key (or to help servers
decide, if they have trouble with determining the default, for some reason).


### Process `Content-Encoding` and decrypt the response

Process all codings in the response's `Content-Encoding` header, in order.
When you encounter the `ewp-rsa-aes128gcm` coding, decrypt it as described
in the [`ewp-rsa-aes128gcm` Encryption Specs][encr-spec].


Security considerations
-----------------------

### When *not* to use this method

 * When the response doesn't contain any private user data, so there's no real
   gain in encrypting it.

 * When you use (and trust) TLS for transport, and the server's internal
   network (the one between a TLS terminator and the actual endpoint) can be
   fully trusted.


### Main security questions

The [Authentication and Security][sec-intro] document
[requires][sec-method-rules] each response encryption method to explicitly
answer a couple of questions:

> How the client's request must look like? How can the server know, that the
> client *wants the server* to use this particular method of encryption?

The client indicates that by including a proper coding (`ewp-rsa-aes128gcm`) in
its `Accept-Encoding` request header.

> How the client delivers his encryption key to the server?

In case of anonymous clients, via the `Accept-Response-Encryption-Key` header.
In case of clients which have already authenticated themselves with an RSA
key-pair, the same key-pair will be used (by default) for response encryption.

> How to encrypt and decrypt the response? Which parts are covered by the
> encryption and which are not?

Encryption and decryption is described above. Only the body of the response is
encrypted. Headers (in particular, the `Content-Type` header) remain
unencrypted.


[develhub]: http://developers.erasmuswithoutpaper.eu/
[statuses]: https://github.com/erasmus-without-paper/ewp-specs-management/blob/stable-v1/README.md#statuses
[sec-intro]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro
[sec-method-rules]: https://github.com/erasmus-without-paper/ewp-specs-sec-intro#rules
[accept-encoding-rfc]: https://tools.ietf.org/html/rfc7231#section-5.3.4
[content-encoding-rfc]: https://tools.ietf.org/html/rfc7231#section-3.1.2.2
[encr-spec]: https://github.com/erasmus-without-paper/ewp-specs-sec-rsa-aes128gcm
[resencr-tls]: https://github.com/erasmus-without-paper/ewp-specs-sec-resencr-tls
[cliauth-httpsig]: https://github.com/erasmus-without-paper/ewp-specs-sec-cliauth-httpsig
