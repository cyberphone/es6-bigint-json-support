# JSON support for BigInt in ES6
_ES6 has recently been upgraded to support a native BigInt type. Currently there is
no explict support for using BigInt with the ES JSON object.
This document contains a proposal for extending the ES6 platform to support BigInt both according to
the JSON standard for numeric data, as well as existing practices relying on JSON strings.
Since JSON do not distinguish between different numbers (aka weakly typed), the described deserialization
schemes all presume that a JSON consumer honors the "contract" including serialization method used by the producer._

# 1 Default Mode
The current ES6 implementation throws an exception if you try to serialize a `BigInt` using `JSON.stringify()`.  This specification _recommends keeping this behavior_ for numerous reasons including:
- Quite _diverging views_ on what the "right" serialization solution is
- Changing default serialization to use JSON Number would give unexpected/unwanted results
- Already _widely deployed_ systems using custom `BigInt` serialization (base64/hex), also including 
current IETF & W3C standards defining JSON structures holding `BigInt` objects
- The tc39 dismissal of the scheme used for `Date`
- The availability of a `BigInt.prototype.toJSON()` option which greatly simplifies customized serialization

# 2 RFC Mode

RFC mode denotes the number serialization scheme specified by the [JSON](https://tools.ietf.org/html/rfc8259) RFC.

## 2.1 JSONNumber Primitive
This proposal builds on the introduction of a new primtive type called `JSONNumber` which is utilized both
for serialization and deserialization. It is only a thin wrapper holding a string in proper
JSON Number notation.  It is recognized by `typeof` as **"jsonnumber"**.

To enable the new functionality
`JSON.stringify()` must be updated to recognize `JSONNumber` as a valid data type which always serializes
verbatim as a string but _without_ quotes.

In the deserializing mode `JSONNumber`can also be used for verifying
that a number actually has expected syntax (in the current `JSON.parse()`
implementation there is no possibility distinguishing between `10` or `10.0`).

JSONNumber is intended to be usable "as is" with a future `BigNum` type,
including when only supplied as a "polyfill".

Programming interface:
<table>
  <tr><th>Constructor</th><td><code>JSONNumber(String)</code></td><td>Initialize JSONNumber</td></tr>
<tr><th>Method</th><td><code>toString()</code></td><td>Get current value</td></tr>
</table>

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

## 2.2 RFC Mode Dserialization
Deserialization of BigInt cannot be automated like serialization,
it depends on an agreement between the consumer and the producer.
This is either performed through the `JSON.parse()` `reviver` option
or is performed after parsing has completed.
Below is an example of a very simple contract:
```js
JSON.parse('{"big":55,"small":55}', 
  (k,v) => typeof v === 'jsonnumber' ? k == 'big' ? BigInt(v.toString()) : Number(v.toString()) : v,
  true   // New flag to make all numbers be returned as JSONNumber
);
```
Expected result: `{big: 55n, small: 55}`

# 3 Quoted String Mode
Although not the method suggested by the JSON RFC, there are quite few systems relying
on `BigInt` objects being represented as JSON Strings.  Unfortunately this practice comes in many flavors
making a standard solution out of reach, or at least not particularly useful. However, there is
no real problem to solve either since _the JSON API as it stands can cope with any variant_.
 
## 3.1 Quoted String Serialization
Here follows a few examples on how to deal with quoted string serialization for `BigInt`.
 
### 3.1.1 Make BigInt by default serialize as decimal digits in quoted strings
 
```js
BigInt.prototype.toJSON = function() { 
  return this.toString(); 
}
 
JSON.stringify({big: 555555555555555555555555555555n, small:55});
```
Expected result: `'{"big":"555555555555555555555555555555","small":55}'`
 
### 3.1.2 Make BigInt by default serialize as Base64Url-encoded data in quoted strings
 
```js
// Browser specific solution
BigInt.prototype.toJSON = function() {
  hex2bin = function(c) {
    return c - (c < 58 ? 48 : 87);
  };
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
  if (!sign && binary[0] > 127) {
    let binp1 = new Uint8Array(binary.length + 1);
    binp1[0] = 0;
    for (q = 0; q < binary.length; q++) {
      binp1[q + 1] = binary[q];
    }
    binary = binp1;
  }
  let text = '';
  for (q = 0; q < binary.length; q++) {
    text += String.fromCharCode(binary[q]);
  }
  return window.btoa(text)
    .replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,'');
}

JSON.stringify({big: 555555555555555555555555555555n, small:55});
 ```
Expected result: `'{"big":"BwMYyOV8edmCI4444w","small":55}'`

_Note: this code is lengthy, complex and potentially incorrect_. There should be a `BigInt` method returning
a byte array in two-complement format like in Java:
https://docs.oracle.com/javase/8/docs/api/java/math/BigInteger.html#toByteArray--

## 3.2 Quoted String Deserialization

Since the is no generally accepted method for adding type information to data embedded in strings,
the selection of encoding method is effectively left to the developers of the actual JSON
ecosystem.  This is either performed through the `JSON.parse()` `reviver` option
or performed after parsing has completed.
 
Here follows a few examples on how to deal with quoted string deserialization for `BigInt`.
 
### 3.2.1 Deserialization of BigInt in quoted strings holding decimal digits
 
```js
JSON.parse('{"big":"55","small":55}', 
  (k,v) => k == 'big' ? BigInt(v) : v
);
```
Expected result: `{big: 55n, small: 55}`
 
### 3.2.2 Deserialization of BigInt serialize in quoted strings holding Base64Url-encoded data
 
```js
// Browser specific solution
function base64Url2BigInt (b64) {
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

_Note: this code is lengthy, complex and potentially incorrect_. There should be a `BigInt` method returning
for creating a value from a byte array in two-complement format like in Java:
https://docs.oracle.com/javase/8/docs/api/java/math/BigInteger.html#BigInteger-byte:A-
