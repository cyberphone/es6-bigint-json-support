# JSON support for BigInt in ES6
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
- The availability of a `BigInt.prototype.toJSON()` option which greatly simplifies customized serialization

## Quoted String Serialization
Although not the method suggested by the JSON RFC, there are quite few systems relying
on `BigInt` objects being represented as JSON Strings.  Unfortunately this practice comes in many flavors
making a standard solution out of reach, or at least not particularly useful. However, there is
no real problem to solve either since _the JSON API as it stands can cope with any variant_.
 
Here follows a few examples on how to deal with quoted string serialization for `BigInt`.
 
### Make BigInt by default serialize as quoted strings
 
```js
BigInt.prototype.toJSON = function() { 
  return this.toString(); 
}
 
JSON.stringify({big: 555555555555555555555555555555n, small:55});
```
Expected result: `'{"big":"555555555555555555555555555555","small":55}'`
 
### Make BigInt by default serialize as Base64Url-encoded quoted strings
 
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
  let binary = '';
  let q = 0;
  while(q < hex.length) {
     let byte = (hex2bin(hex.charCodeAt(q++)) << 4) + hex2bin(hex.charCodeAt(q++));
     if (q == 0) {
       if (sign) {
         mask = 128;
         while (byte & mask == 0) {
           byte |= mask;
           byte >= 1;
         }
       } else {
         if (byte > 127) binary = String.fromCharCode(0);
       }
     }
     binary += String.fromCharCode(byte);
  }
  return window.btoa(binary)  // Not yet verified code...
    .replace(/\+/g,'-').replace(/\//g,'_').replace(/=/g,'');
}

JSON.stringify({big: 555555555555555555555555555555n, small:55});
 ```
Expected result: `'{"big":"BwMYyOV8edmCI4444w","small":55}'`

## Quoted String Deserialization

Since the is no generally accepted method for adding type information to data embedded in strings,
deserialization (parsing) is effectively left to developers who must honor the "contract"
used by the producer.  This is either performed through the `JSON.parse()` `reviver` option
or performed after parsing has completed.
 
Here follows a few examples on how to deal with quoted string deserialization for `BigInt`.
 
### Make BigInt by default serialize as quoted strings
 
```js
BigInt.prototype.toJSON = function() { 
  return this.toString(); 
}
 
JSON.stringify({big: 555555555555555555555555555555n, small:55});
```
Expected result: `'{"big":"555555555555555555555555555555","small":55}'`
 
### Make BigInt by default serialize as Base64Url-encoded quoted strings
 
```js
// Browser specific solution
BigInt.prototype.toJSON = function() {
  return window.btoa(this.getBytes(true))  // Not yet verified code...
    .replace(/\+/g,'-').replace(/\//g,'_');
}

JSON.stringify({big: 555555555555555555555555555555n, small:55});
 ```
Expected result: `'{"big":"BwMYyOV8edmCI4444w","small":55}'`
