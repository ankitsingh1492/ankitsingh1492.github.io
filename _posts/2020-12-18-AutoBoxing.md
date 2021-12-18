---
title: "JavaScript Autoboxing"
published: true
---

JS has following primitives - 

-  string
-  number
-  bigint
-  boolean
-  undefined
-  symbol
-  null.

Ever wondered how are we able to call methods like toString() etc. on primitive values ? As, they are not like our regular objects in JS.

## Example from The principles of Object-Oriented JS by Nicholas C. Zakas

```js
let name = "Ankit";
let firstChar = name.charAt(0);
console.log(firstChar); // "A"
```
Here we can see that we called charAt on name which is assigned a primitive value string. The second line treats name like an object and calls charAt(0) using dot notation.

Below is what happens behind the scenes or at the JS engine level - 
```js
let name = "Ankit";
let temp = new String(name);
let firstChar = temp.charAt(0);
temp = null;
console.log(firstChar) // "A"
```

The engine creates an instance of a String, thus the charAt(0) works. The object exists only for one statement before it's destroyed. 

## Note
Except for null and undefined, all primitive values have object equivalents that wrap around the primitive values:

- String for the string primitive.
- Number for the number primitive.
- BigInt for the bigint primitive.
- Boolean for the boolean primitive.
- Symbol for the symbol primitive.
- The wrapper's valueOf() method returns the primitive value.

## Resources

- [book] - The principles of Object-oriented JS - Nicholas C. Zakas
- [Mozilla] - MDN docs

[//]: # (These are reference links used in the body of this note and get stripped out when the markdown processor does its job. There is no need to format nicely because it shouldn't be seen. Thanks SO - http://stackoverflow.com/questions/4823468/store-comments-in-markdown-syntax)

   [book]: https://www.amazon.in/Principles-Object-Oriented-JavaScript-Nicholas-Zakas/dp/1593275404
   [Mozilla]: https://developer.mozilla.org/en-US/docs/Glossary/Primitive