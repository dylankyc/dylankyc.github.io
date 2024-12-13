# Narrowing

<!-- toc -->

In TypeScript, **narrowing** is the process of refining a variable's type from a broader type to a more specific one based on certain runtime checks. TypeScript uses control flow analysis to track the execution paths in your code, allowing it to infer more specific types in different branches. Here are some common techniques for narrowing:

### 1. `typeof` Type Guards

You can use the `typeof` operator to check the basic data type of a variable:

```typescript
if (typeof value === "string") {
  // TypeScript knows that value is a string here
}
```

### 2. Truthiness Checking

In conditional statements, TypeScript narrows types based on truthiness. For example:

```typescript
if (value) {
  // TypeScript knows value is not null, undefined, 0, NaN, or an empty string here
}
```

### 3. Equality Narrowing

By comparing two values for equality, TypeScript can infer that they are of the same type:

```typescript
if (x === y) {
  // x and y are narrowed to the same type here
}
```

### 4. `in` Operator Narrowing

The `in` operator checks if an object has a property, which can narrow the type:

```typescript
if ("property" in obj) {
  // obj has the property here
}
```

### 5. `instanceof` Narrowing

The `instanceof` operator checks whether an object is an instance of a particular class, narrowing the type:

```typescript
if (obj instanceof SomeClass) {
  // obj is known to be an instance of SomeClass here
}
```

### 6. Control Flow Analysis

TypeScript analyzes the control flow of your code, allowing it to track the types of variables. This means that a variable's type can be narrowed in different branches of code:

```typescript
function example(value: number | string) {
  if (typeof value === "number") {
    // value is narrowed to number here
    return value.toFixed(2);
  }
  // value is narrowed to string here
  return value.toUpperCase();
}
```

### 7. User-Defined Type Guards

You can define a function that returns a type predicate to narrow types more flexibly:

```typescript
function isFish(pet: Fish | Bird): pet is Fish {
  return (pet as Fish).swim !== undefined;
}

if (isFish(pet)) {
  pet.swim(); // pet is inferred to be Fish here
}
```

These narrowing techniques allow TypeScript to provide more stringent type checking at compile time, reducing runtime errors and enhancing the safety of your code.
