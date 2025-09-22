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

## Synchronization Questions

1. **Partial matching**

In the generate sync, the Request.shortenUrl action only includes the shortUrlBase because that is the only piece of information needed to trigger the generation of a unique nonce for that specific context. The targetUrl is not relevant for this step. In the register sync, both shortUrlBase and targetUrl are required because this sync is responsible for creating the actual association between the newly generated nonce (the suffix), the base URL, and the final target URL in the UrlShortening concept.


2. **Omitting names**

The convention of omitting names when the argument or result name is the same as the variable name is not used in every case to avoid ambiguity and maintain clarity. For instance, in the register sync, shortUrlSuffix: nonce is explicitly named because the register action's argument is shortUrlSuffix, while the variable holding the value is nonce. If we simply wrote nonce, it would be unclear which argument of the register action it corresponds to. This would have been even more unclear if there were 2 such variables without their explanation. That is why, explicit naming is essential when the variable name and the action's parameter name differ.

3. **Inclusion of request**

The Request.shortenUrl action is included in the first two syncs because they are directly triggered by a user's request to create a short URL. The setExpiry sync, however, is triggered by an internal system event: the completion of the UrlShortening.register action. It's a subsequent, automatic action that happens as a consequence of a shortening being created, not directly as a result of the initial user request.

4. **Fixed domain**

We can hardcode the fixed domain name so that Request.shortenUrl will no longer need the shortUrlBase argument. Let's assume we want to use the fixed domain "bit.ly":

```
  sync generate
  when Request.shortenUrl ()
  then NonceGeneration.generate (context: "bit.ly")

  sync register
  when
    Request.shortenUrl (targetUrl)
    NonceGeneration.generate (): (nonce)
  then UrlShortening.register (shortUrlSuffix: nonce, "bit.ly", targetUrl)

  sync setExpiry
  when UrlShortening.register (): (shortUrl)
  then ExpiringResource.setExpiry (resource: shortUrl, seconds: 3600)
```

5. **Adding a sync**

```
sync expireShortUrl
when ExpiringResource.system.expireResource (): (resource: shortUrl)
  then UrlShortening.delete (shortUrl: shortUrl)
```

## Extending the design

