# Algorithm Lucidity

Algorithm Lucidity refers to resilience against algorithm confusion attacks.

This document aims to make it easy for PASETO implementations to achieve this property.

## PASETO Cryptography Key Requirements

Cryptography keys in PASETO are defined as **both the raw key material and its
parameter choices, not just the raw key material**.

PASETO implementations **MUST** enforce some logical separation between different key types;
especially when the raw key material is the same (i.e. a 256-bit opaque blob).

Arbitrary strings (or byte arrays, or equivalent language constructs) **MUST NOT**
be accepted as a key in any PASETO library, **UNLESS** it's an application-specific
encoding that encapsulates both the key and an algorithm identifier. (For example,
a [`k2.local` PASERK](https://github.com/paseto-standard/paserk/blob/master/types/local.md).)

## Implementation Guidance

### Object-Oriented Languages

Strict type separation should be employed in object-oriented languages.

* For local tokens, you only need a `SymmetricKey`.
* For public tokens, you need an `AsymmetricSecretKey` and an `AsymmetricPublicKey`.

Each key type should be parametrized by two inputs: The key material and an algorithm identifier.

For example, you might implement something like this:

```java
public enum Version { V1, V2, V3, V4 };
public enum Purpose { PURPOSE_LOCAL, PURPOSE_PUBLIC }; 

absract class Key {
     protected byte[] material;
     protected Version version;
     
     public Key(byte[] keyMaterial, Version version) {
     }
     
     abstract public bool isKeyValidFor(Version v, Purpose p);
     
     /* ... */
}

class SymmetricKey extends Key {
    public SymmetricKey(byte[] keyMaterial, Version version) {
        super(keyMaterial, version);
    }
    
    public bool isKeyValidFor(Version v, Purpose p) {
        return v == this.version && p == Purpose.PURPOSE_LOCAL;
    }
}
class AsymmetricSecretKey extends Key {
    public SymmetricKey(byte[] keyMaterial, Version version) {
        super(keyMaterial, version);
    }
    
    public bool isKeyValidFor(Version v, Purpose p) {
        return v == this.version && p == Purpose.PURPOSE_PUBLIC;
    }
}
class AsymmetricPublicKey extends Key {
    public SymmetricKey(byte[] keyMaterial, Version version) {
        super(keyMaterial, version);
    }
    
    public bool isKeyValidFor(Version v, Purpose p) {
        return v == this.version && p == Purpose.PURPOSE_PUBLIC;
    }
}
```

Whenever you implement one of the PASETO operations, you would then first invoke
`givenKey.isKeyValidFor()` for the expected version and purpose. If it doesn't
return true, throw an `Exception`.

### Procedural Languages

If you're working in a procedural programming language, where the concept of classes isn't
available to lean on, the algorithm identifier should be hard-coded alongside the key material
(e.g. in a `struct`) and checked by the library.

## Why This Matters

PASETO largely obviates the risk of algorithm confusion attacks by design. It accomplishes
this by limiting the in-band negotiation to the bare minimum:

* What version are you using?
* Are you expecting a `local` or `public` token?

However, PASETO implementations that allow the incorrect key to be used for a given algorithm
may open themselves up to attack.