# JSON support for BigInt in ES6
_ES6 has recently been upgraded to support a native BigInt type. Currently there is
no explicit support for using BigInt with the ES6 JSON object.
This document contains a proposal for extending the ES6 platform to support BigInt both according to
the JSON standard for numeric data, as well as existing practices relying on JSON strings.
Since JSON do not distinguish between different numbers (aka weakly typed), the described deserialization
schemes all presume that a JSON consumer honors an in advance known "contract" including serialization method used by the producer._

Also see [summary of changes](#summary-of-changes).

# 1 Default Mode
The current ES6 implementation throws an exception if you try to serialize a `BigInt` using `JSON.stringify()`.  This specification _recommends keeping this behavior_ for numerous reasons including:
- _Diverging views_ on what the "right" serialization solution is
- Changing default serialization to use JSON Number would give unexpected/unwanted results
- _Widely deployed_ systems relying on custom `BigInt` serialization (base64/hex), also including 
current IETF & W3C standards defining JSON structures holding `BigInt` objects
- TC39's dismissal of the serialization scheme used for `Date`
- Availability of a `BigInt.prototype.toJSON()` option which greatly simplifies customized serialization

# 2 RFC Mode

RFC mode denotes the number serialization scheme specified by the [JSON](https://tools.ietf.org/html/rfc8259) RFC.

## 2.1 The <code>JSONNumber</code> Primitive
This proposal builds on the introduction of a new primitive type called `JSONNumber`, intended
for serialization and deserialization of numeric data which cannot be represented by
the ES6 `Number` type. `JSONNumber` is a thin wrapper holding a string in proper
[JSON Number notation](https://tools.ietf.org/html/rfc8259#section-6).  The `typeof` operator recognizes `JSONNumber` as **"jsonnumber"**.

To enable the new functionality
`JSON.stringify()` must be updated to recognize `JSONNumber` as a valid data type which always serializes
verbatim as a string but _without_ quotes.

In the deserializing mode `JSONNumber` can also be used for verifying
that a number actually has expected syntax (in the current `JSON.parse()`
implementation there is no possibility distinguishing between `10` or `10.0`).

`JSONNumber` is intended to be usable "as is" with a future `BigNum` type,
including when only supplied as a "polyfill".

### 2.1.1 Interface
<table>
  <tr><th>Method</th><th>Comment</th></tr>
  <tr><td><code>JSONNumber(</code><i>string</i><code>)</code></td><td>Constructor.  The <i>string</i> argument is
    supposed to already be in proper JSON&nbsp;Number notation</td></tr>
  <tr><td><code>JSONNumber.prototype.toString()</code></td><td>Returns <i>string</i> value</td></tr>
  <tr><td><code>JSONNumber.prototype.isInteger()</code></td><td>Returns <b>true</b> for integer syntax
    (=<i>string</i> contains no decimal point or exponent)</td></tr>
  <tr><td><code>JSONNumber.prototype.isPositive()</code></td><td>Returns <b>true</b> for positive numbers</td></tr>
  <tr><td><code>JSONNumber.prototype.isNumber()</code></td><td>Returns <b>true</b> if a converted <i>string</i> would
    be 100% compatible with an ES6 <code>Number</code> with respect to precision and range</td></tr>
</table>

_Question_: Since `JSONNumber` does not seem to have any logical use outside of the ES6 `JSON`object, would
it be possible (and useful) making `JSONNumber` a member like `JSON.JSONNumber`?

## 2.1 RFC Mode Serialization
The following code shows how RFC mode `BigInt` serialization can be added
to `JSON.stringify`.
```js
BigInt.prototype.toJSON = function() { 
  return JSONNumber(this.toString()); 
}
 
JSON.stringify({big: 555555555555555555555555555555n, small:55});
```
Expected result: `'{"big":555555555555555555555555555555,"small":55}'`

**NOTE:** RFC mode support requires `JSON.stringify()` to be upgraded to accept `JSONNumber` as a serializable object.

## 2.2 RFC Mode Deserialization
Deserialization of `BigInt` cannot be automated like serialization;
the selection between different number types usually
depends on conventions between JSON consumers and producers.
The selections are either managed through the `JSON.parse()` `reviver` option
or are performed after parsing has completed.

**NOTE:** RFC mode deserialization requires a new _optional_ flag to `JSON.parse()`.  When this flag is set to **true**,
JSON Number elements must only be parsed for correctness with respect to syntax, while the
parsed string itself is returned in a `JSONNumber` for application level processing.

### 2.2.1 Property Based Deserialization Selection
Below is an example of a scheme having a single property holding a `BigInt`:
```js
JSON.parse('{"big":55,"small":55}', 
  (k,v) => typeof v === 'jsonnumber' ? k == 'big' ? BigInt(v.toString()) : Number(v.toString()) : v,
  true   // New flag to make all numbers be returned as JSONNumber
);
```
Expected result: `{big: 55n, small: 55}`

### 2.2.2 Value Based Deserialization Selection
Below is an example where the actual value of an object is used for type selection:
```js
JSON.parse('{"big":555555555555555555555555555555,"small":55}', 
  (k,v) => typeof v === 'jsonnumber' ? v.isNumber() ? Number(v.toString()) : BigInt(v.toString()) : v,
  true   // New flag to make all numbers be returned as JSONNumber
);
```
Expected result: `{big: 555555555555555555555555555555n, small: 55}`

### 2.2.3 Syntax Checking Deserialization
Below is an example of a syntax checker using `JSONNumber`:
```js
JSON.parse('{"int1":55,"int2":10.0}', 
  (k,v) => {
     if (typeof v === 'jsonnumber') {
       if (!v.isInteger()) {
         throw new Error('Not integer: ' + k);
       }
       return Number(v.toString());
     }
     return v;
  },
  true   // New flag to make all numbers be returned as JSONNumber
);
```
Expected result: An error message containing the name `int2`;

# 3 Quoted String Mode
Although not the method suggested by the JSON RFC, there are quite few systems relying
on `BigInt` objects being represented as JSON Strings.  Unfortunately this practice comes in many flavors
making a standard solution out of reach, or at least not particularly useful. However, there is
no real problem to solve either since _the ES6 JSON API as it stands can cope with any variant_.
 
## 3.1 Quoted String Serialization
Here follows a few examples on how to deal with quoted string serialization for `BigInt`.
 
### 3.1.1 Serialization Using Decimal Digits
 
```js
BigInt.prototype.toJSON = function() { 
  return this.toString(); 
}
 
JSON.stringify({big: 555555555555555555555555555555n, small:55});
```
Expected result: `'{"big":"555555555555555555555555555555","small":55}'`
 
### 3.1.2 Serialization Using Base64Url Encoded Data
 
```js
// Browser specific solution
BigInt.prototype.toJSON = function() {
  function hex2bin(c) {
    return c - (c < 58 ? 48 : 87); 
  }
  let v = this.valueOf();
  let sign = false;
  if (v < 0) {
    v = -v;
    sign = true;
  }
  let hex = v.toString(16);
  if (hex.length & 1) hex = '0' + hex;
  let binary = new Uint8Array(hex.length / 2);
  let i = binary.length;
  let q = hex.length;
  let carry = 1;
  while(q > 0) {
     let byte = hex2bin(hex.charCodeAt(--q)) + (hex2bin(hex.charCodeAt(--q)) << 4);
     if (sign) {
       byte = ~byte + carry;
       if (byte > 255) {
         carry = 1;
       } else {
         carry = 0;
       }
     }
     binary[--i] = byte;
  }
  if (sign ^ (binary[0] > 127)) {
    let binp1 = new Uint8Array(binary.length + 1);
    binp1[0] = sign ? 255 : 0;
    for (q = 0; q < binary.length; q++) {
      binp1[q + 1] = binary[q];
    }
    binary = binp1;
  }
  let text = '';
  for (q = 0; q < binary.length; q++) {
    text += String.fromCharCode(binary[q]);
  }
  return window.btoa(text).replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,'');
}

JSON.stringify({big: 555555555555555555555555555555n, small:55});
 ```
Expected result: `'{"big":"BwMYyOV8edmCI4444w","small":55}'`

**NOTE:** _This code is lengthy, complex and potentially incorrect_. There should be a `BigInt` method returning
a byte array in two-complement format like in Java:
https://docs.oracle.com/javase/8/docs/api/java/math/BigInteger.html#toByteArray--

## 3.2 Quoted String Deserialization

Since the is no generally accepted method for adding type information to data embedded in strings,
the selection of encoding method is effectively left to the developers of the actual JSON
ecosystem.  The selections are either managed through the `JSON.parse()` `reviver` option
or are performed after parsing has completed.
 
Here follows a few examples on how to deal with quoted string deserialization for `BigInt`.
 
### 3.2.1 Deserialization Using Decimal Digits
 
```js
JSON.parse('{"big":"55","small":55}', 
  (k,v) => k == 'big' ? BigInt(v) : v
);
```
Expected result: `{big: 55n, small: 55}`
 
### 3.2.2 Deserialization Using Base64Url Encoded Data
 
```js
// Browser specific solution
function base64Url2BigInt(b64) {
  let dec = window.atob(b64.replace(/\-/g,'+').replace(/_/g,'/').replace(/=/g,'?'));
  let binary = new Uint8Array(dec.length);
  for (let q = 0; q < binary.length; q++) {
    binary[q] = dec.charCodeAt(q);
  }
  let sign = false;
  if (binary[0] > 127) {
    sign = true;
    let carry = 1;
    let q = binary.length; 
    while (q > 0) {
      let byte = ~binary[--q] + carry;
      binary[q] = byte;
      if (byte > 255) {
        carry = 1;
      } else {
        carry = 0;
      }
    }
  }
  let v = BigInt(0n);
  for (let q = 0; q < binary.length; q++) {
    v *= 256n;
    v += BigInt(binary[q]);
  }
  if (sign) {
     return -v;
  }
  return v;
}

JSON.parse('{"big":"BwMYyOV8edmCI4444w","small":55}', 
  (k,v) => k == 'big' ? base64Url2BigInt(v) : v
);

```
Expected result: `{big: 555555555555555555555555555555n, small: 55}`

**NOTE:** _This code is lengthy, complex and potentially incorrect_. There should be a method
for creating a `BigInt` value from a byte array in two-complement format like in Java:
https://docs.oracle.com/javase/8/docs/api/java/math/BigInteger.html#BigInteger-byte:A-

# Summary Of Changes
- Adding a `JSONNumber` primitive type
- Enhancing `JSON.stringify()` to always accept `JSONNumber` for serialization
- Adding an optional flag to `JSON.parse()` requiring the parsing process to return `JSONNumber` instead of `Number`
- _Optionally_ improving `BigInt` for dealing with two complement serialization formats 

### Aknowledgements
This specification was influenced by input from many persons including
Richard Gibson, Kai Zhu, Jordan Harband, Rob Ede, T.J. Crowder, Daniel Ehrenberg,
Michael Theriot, Claude Pache, Ranando King, J Decker, Kevin Gibbons,
Claude Petit, Jakob Kummerow and Isiah Meadows.

### Current Version
0.1
