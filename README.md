# Metadata Reflection API

* [Detailed proposal][Metadata-Spec]

## Installation

```
npm install reflect-metadata
```

## Background

* Decorators add the ability to augment a class and its members as the class is defined, through a declarative syntax.
* Traceur attaches annotations to a static property on the class.
* Languages like C# (.NET), and Java support attributes or annotations that add metadata to types, along with a reflective API for reading metadata.

## Goals

* A number of use cases (Composition/Dependency Injection, Runtime Type Assertions, Reflection/Mirroring, Testing) want the ability to add additional metadata to a class in a consistent manner.
* A consistent approach is needed for various tools and libraries to be able to reason over metadata.
* Metadata-producing decorators (nee. "Annotations") need to be generally composable with mutating decorators.
* Metadata should be available not only on an object but also through a Proxy, with related traps.
* Defining new metadata-producing decorators should not be arduous or over-complex for a developer.
* Metadata should be consistent with other language and runtime features of ECMAScript.

## Syntax

* Declarative definition of metadata:
```JavaScript
class C {
  @Reflect.metadata(metadataKey, metadataValue)
  method() {
  }
}
```

* Imperative definition of metadata:
```JavaScript
Reflect.defineMetadata(metadataKey, metadataValue, C.prototype, "method");
```

* Imperative introspection of metadata:
```JavaScript
let obj = new C();
let metadataValue = Reflect.getMetadata(metadataKey, obj, "method");
```

## Semantics

* Object has a new \[\[Metadata\]\] internal property that will contain a Map whose keys are property keys (or **undefined**) and whose values are Maps of metadata keys to metadata values.
* Object will have a number of new internal methods for \[\[DefineOwnMetadata\]\], \[\[GetOwnMetadata\]\], \[\[HasOwnMetadata\]\], etc.
  * These internal methods can be overridden by a Proxy to support additional traps.
  * These internal methods will by default call a set of abstract operations to define and read metadata.
* The Reflect object will expose the MOP operations to allow imperative access to metadata.
* Metadata defined on class declaration *C* is stored in *C*.\[\[Metadata\]\], with **undefined** as the key.
* Metadata defined on static members of class declaration *C* are stored in *C*.\[\[Metadata\]\], with the property key as the key.
* Metadata defined on instance members of class declaration *C* are stored in *C*.prototype.\[\[Metadata\]\], with the property key as the key.

## API

```JavaScript
// define metadata on an object or property
Reflect.defineMetadata(metadataKey, metadataValue, target);
Reflect.defineMetadata(metadataKey, metadataValue, target, propertyKey);

// check for presence of a metadata key on the prototype chain of an object or property
let result = Reflect.hasMetadata(metadataKey, target);
let result = Reflect.hasMetadata(metadataKey, target, propertyKey);

// check for presence of an own metadata key of an object or property
let result = Reflect.hasOwnMetadata(metadataKey, target);
let result = Reflect.hasOwnMetadata(metadataKey, target, propertyKey);

// get metadata value of a metadata key on the prototype chain of an object or property
let result = Reflect.getMetadata(metadataKey, target);
let result = Reflect.getMetadata(metadataKey, target, propertyKey);

// get metadata value of an own metadata key of an object or property
let result = Reflect.getOwnMetadata(metadataKey, target);
let result = Reflect.getOwnMetadata(metadataKey, target, propertyKey);

// get all metadata keys on the prototype chain of an object or property
let result = Reflect.getMetadataKeys(target);
let result = Reflect.getMetadataKeys(target, propertyKey);

// get all own metadata keys of an object or property
let result = Reflect.getOwnMetadataKeys(target);
let result = Reflect.getOwnMetadataKeys(target, propertyKey);

// delete metadata from an object or property
let result = Reflect.deleteMetadata(metadataKey, target);
let result = Reflect.deleteMetadata(metadataKey, target, propertyKey);

// apply metadata via a decorator to a constructor
@Reflect.metadata(metadataKey, metadataValue)
class C {
  // apply metadata via a decorator to a method (property)
  @Reflect.metadata(metadataKey, metadataValue)
  method() {
  }
}
```

## Alternatives

* Use properties rather than a separate API.
  * Obvious downside is that this can be a lot of code:
```JavaScript
function ParamTypes(...types) {
  return (target, propertyKey) => {
    const symParamTypes = Symbol.for("design:paramtypes");
    if (propertyKey === undefined) {
      target[symParamTypes] = types;
    }
    else {
      const symProperties = Symbol.for("design:properties");
      let properties, property;
      if (Object.prototype.hasOwnProperty.call(target, symProperties)) {
        properties = target[symProperties];
      }
      else {
        properties = target[symProperties] = {};
      }
      if (Object.prototype.hasOwnProperty.call(properties, propertyKey)) {
        property = properties[propertyKey];
      }
      else {
        property = properties[propertyKey] = {};
      }
      property[symParamTypes] = types;
    }
  };
}
```

## Notes
* Though it may seem counterintuitive, the methods on Reflect place the parameters for the metadata key and metadata value before the target or property key. This is due to the fact that the property key is the only optional parameter in the argument list. This also makes the methods easier to curry with Function#bind. This also helps reduce the overall footprint and complexity of a metadata-producing decorator that could target both a class or a property:

```JavaScript
function ParamTypes(...types) {
  // as propertyKey is effectively optional, its easier to use here
  return (target, propertyKey) => { Reflect.defineMetadata("design:paramtypes", types, target, propertyKey); }

  // vs. having multiple overloads with the target and key in the front:
  //
  // return (target, propertyKey) => {
  //    if (propertyKey === undefined) {
  //      Reflect.defineMetadata(target, "design:paramtypes", types);
  //    }
  //    else {
  //      Reflect.defineMetadata(target, propertyKey, "design:paramtypes", types);
  //    }
  // }
  //
  // vs. having a different methods for the class or a property:
  //
  // return (target, propertyKey) => {
  //    if (propertyKey === undefined) {
  //      Reflect.defineMetadata(target, "design:paramtypes", types);
  //    }
  //    else {
  //      Reflect.definePropertyMetadata(target, propertyKey, "design:paramtypes", types);
  //    }
  // }
}
```

* To enable experimental support for metadata decorators in your TypeScript project, you must add `"experimentalDecorators": true` to your tsconfig.json file.
* To enable experimental support for auto-generated type metadata in your TypeScript project, you must add `"emitDecoratorMetadata": true` to your tsconfig.json file.
  * Please note that auto-generated type metadata may have issues with circular or forward references for types.

### Using `emitDecoratorMetadata` with SES (Secure ECMAScript)

SES does not permit mutation of globals. To use `reflect-metadata` in an SES host, you must import it using `reflect-metadata/no-conflict`:

```ts
// --module es2015
// --module es2020
// --module esnext
import { Reflect } from "reflect-metadata/no-conflict";
...
```

If your `--module` target is not an ES module target, you must create a local alias for the imported binding as TypeScript will rewrite the import for CommonJS/AMD/System/etc.:

```ts
// --module commonjs
// --module amd
// --module umd
// --module system
import { Reflect as _Reflect } from "reflect-metadata/no-conflict";
const Reflect = _Reflect;

// which, for 'commonjs', is transformed to:
const no_conflict_1 = require("reflect-metadata/no-conflict");
const Reflect = no_conflict_1.Reflect;
```

The manual local alias for `Reflect` is necessary as the TypeScript `__metadata` helper specifically looks for `Reflect` as if it were a global.

**NOTE:** This will *not* work with `--importHelpers` unless you redirect `tslib` to a custom copy that loads `"reflect-metadata/no-conflict"`. More information on how to support this scenario is forthcoming.

## Issues

* A poorly written mutating decorator for a class constructor could cause metadata to become lost if the prototype chain is not maintained. Though, not maintaining the prototype chain in a mutating decorator for a class constructor would have other negative side effects as well. @rbuckton
  * This is mitigated if the mutating decorator returns a class expression that extends from the target, or returns a proxy for the decorator. @rbuckton
* Metadata for a method is attached to the class (or prototype) via the property key. It would not then be available if trying to read metadata on the function of the method (e.g. "tearing-off" the method from the class). @rbuckton

[Metadata-Spec]: https://rbuckton.github.io/reflect-metadata