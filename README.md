# Proposal for JSON support for BigInt in ES6
_ES6 has recently been upgraded to support a native BigInt type. Currently there is
no explict support for using BigInt with the ES JSON object.
This document contains a proposal for extending the ES6 platform to support both current practices
as well as the JSON standard for numeric data._

## Default Mode
The current ES6 implementation throws an exception if you try to serialize a `BigInt` using `JSON.stringify()`.  This specification _recommends keeping this behavior_ for numerous reasons including:
- Quite _diverging views_ on what the "right" serialization solution is
- Changing serialization to use JSON Number by default gives unexpected/unwanted results
- Already _widely deployed_ systems using custom `BigInt` serialization (base64/hex) including current IETF & W3C standards defining JSON structures holding `BigInt` objects
- The tc39 dismissal of the scheme used for `Date`
- The availability of a `toJSON()` option which greatly simplifies customized serialization

 
