# Solana useWallet example

<!-- toc -->

Here's a blog post about TypeScript's type narrowing:

# Understanding TypeScript Type Narrowing: A Real-World Example

When working with TypeScript, you might encounter situations where a variable could be of multiple types. Here's a practical example I encountered while building a Solana dApp that demonstrates how TypeScript's type narrowing can help us write safer code.

## The Problem

In our Solana application, we were using the `useWallet()` hook which returns a `publicKey` that could be either a `PublicKey` object or `null`:

```typescript
const { publicKey } = useWallet(); // Type: PublicKey | null
```

When trying to pass this `publicKey` to a function that specifically required a `PublicKey`, TypeScript complained:

```typescript
// Error: Type 'PublicKey | null' is not assignable to type 'PublicKey'
initialize.mutateAsync({ name: bankName, owner: publicKey });
```

## The Solution: Type Narrowing

TypeScript has a powerful feature called "type narrowing" that helps us handle these situations. By adding a null check, we can narrow the type from `PublicKey | null` to just `PublicKey`:

```typescript
if (!publicKey) {
  toast.error("Please connect your wallet");
  return;
}
// TypeScript now knows publicKey is definitely a PublicKey
initialize.mutateAsync({ name: bankName, owner: publicKey });
```

## How It Works

TypeScript's control flow analysis understands that:

1. If `publicKey` is `null`, the function will return early
2. Therefore, in the code after the check, `publicKey` must be a `PublicKey`
3. This automatically narrows the type, making it safe to use

## Best Practices

This pattern is not just about making TypeScript happy – it's about writing more robust code:

- It handles edge cases explicitly
- It provides clear feedback to users
- It prevents runtime errors that could occur if we tried to use a null value

## Conclusion

Type narrowing is one of TypeScript's most powerful features for writing type-safe code. By understanding and using it effectively, we can catch potential errors at compile time rather than runtime, while also providing better user experiences.

Remember: When dealing with nullable types, always consider using type narrowing to handle all possible cases explicitly. Your future self (and your users) will thank you!
