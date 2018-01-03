Release notes
=============

This document describes all the changes made to the *`ewp-rsa-aes128gcm`
Response Encryption* document, starting from its first released version.


0.3.0
-----

* Clarified, that the server MUST NOT ignore the client's request to encrypt
  the response.
* Clarified what to do in case of some specific errors (e.g. invalid Base64
  content in `Accept-Response-Encryption-Key` header).
* Clarified, that the server is REQUIRED to be able to extract the encryption
  key from the request if HTTP Signature Client Authentication is used, and
  that it is NOT REQUIRED to be able to extract such keys from other client
  authentication methods.
* Similarly, clarified when `Accept-Response-Encryption-Key` is required and
  when not.
* Added the recommendation (for both clients and servers) to support the `gzip`
  encoding.
* Clarified that `Content-Type` must not be obfuscated.


0.2.0
-----

* Switched from AES CBC to AES GCM (see
  [here](https://github.com/erasmus-without-paper/ewp-specs-sec-rsa-aes128cbc/issues/1)).


0.1.0
-----

Initial release.
