# Proposal for JSON support for BigInt in ES6
_ES6 has recently been upgraded to support a native BigInt type. Currently there is
no explict support for using BigInt with the ES JSON object.
This document contains a proposal for extending the ES6 platform to support BigInt both according to
the JSON standard for numeric data, as well as existing practices relying on JSON strings._

## Default Mode
The current ES6 implementation throws an exception if you try to serialize a `BigInt` using `JSON.stringify()`.  This specification _recommends keeping this behavior_ for numerous reasons including:
- Quite _diverging views_ on what the "right" serialization solution is
- Changing default serialization to use JSON Number would give unexpected/unwanted results
- Already _widely deployed_ systems using custom `BigInt` serialization (base64/hex), also including 
current IETF & W3C standards defining JSON structures holding `BigInt` objects
- The tc39 dismissal of the scheme used for `Date`
- The availability of a `BigInt.toJSON()` option which greatly simplifies customized serialization

 ## Quoted String Serialization
 Although not the method suggested by the JSON RFC, there are quite few systems relying
 on `BigInt` objects being represented as JSON Strings.  Unfortunately this practice comes in many flavors
 making a standard solution out of reach, or at least not particularly useful. However, there is
 no real problem to solve either since _the JSON API as it stands can cope with any variant_.
 
 Here follows a few examples on how to deal with quoted string serialization for `BigInt`.
 
 ### Making all BigInts serialize as "nnnnn"
 
 ```js
 BigInt.prototype.toJSON = function() { 
   return this.toString(); 
 }
 ```
 
 ### Making all BigInts serialize as Base64-encoded quoted strings
 
 ```js
 // Browser specific solution
 BigInt.prototype.toJSON = function() {
   return window.btoa(this.getBytes(true));  // Not yet verified code...
 }
 ```
