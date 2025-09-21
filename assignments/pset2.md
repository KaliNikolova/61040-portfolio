# Problem Set 2

## Concepts Questions

1. **Context**

In the NonceGeneration concept, contexts are used to create separate namespaces for generating unique strings. This ensures that a nonce generated for one context will not conflict with a nonce generated for another. In the URL shortening application, a context will correspond to the shortUrlBase (e.g., a specific domain like "tinyurl.com" or a user-specific domain). This allows the service to guarantee that the generated shortUrlSuffix is unique for that particular base URL, preventing collisions.


2. **Storing used strings**

The NonceGeneration concept must store sets of used strings to uphold its principle: "each generate returns a string not returned before for that context." Without a record of previously used strings, it would be impossible to guarantee the uniqueness of a newly generated one.

In an implementation where a counter is used for each context, the set of "used strings" in the specification corresponds to the set of all string representations of the integers from the counter's starting value up to its current value for that context. The abstraction function mapping the implementation (a counter) to the abstract state (a set of used strings) would be: For a given context, the set of used strings is the set of all strings formed by the integer values from the initial counter value to the current counter value minus one.

3. **Words as nonces**

Using common dictionary words for nonce generation offers the following:

Advantage: The resulting short URLs are more memorable and easier for users to type and share verbally (e.g., "yellkey.com/happy").

Disadvantage: The pool of easily memorable words is limited, which could lead to a faster exhaustion of available short URLs compared to using alphanumeric strings. This could also make them more susceptible to guessing by malicious actors.

To modify the NonceGeneration concept, we could introduce a pre-defined set of words and draw from it. The concept could be changed as follows:

```
  concept NonceGeneration [Context]
  purpose generate unique strings within a context from a word list
  principle each generate returns a string not returned before for that context
  state
    a set of Contexts with
      a used set of Strings
      an available set of Strings
  actions
    generate (context: Context) : (nonce: String)
      requires the available set for the context is not empty
      effect returns a nonce from the available set that is not already used by this context; deletes the nonce from available and adds it to used.
```