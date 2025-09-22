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
  then UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase: "bit.ly", targetUrl)

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

1. 

```
concept AnalyticsTracking [Resource]
  purpose count the number of times a resource is accessed
  principle after a resource is created, its access count can be incremented and retrieved
  state
    a set of Resources with
      an accessCount Number
  actions
    create (resource: Resource)
      effect creates an analytics entry for the resource with an initial access count of 0
    increment (resource: Resource)
      effect increments the access count for the given resource by one
    getCount (resource: Resource): (count: Number)
      requires an analytics entry exists for the resource
      effect returns the current access count for the resource

  concept UserOwnership [Resource, User]
  purpose associate a resource with a creating user
  principle a resource is owned by the user who created it
  state
    a set of Resources with
      an owner User
  actions
    assignOwner (resource: Resource, user: User)
      requires the resource does not already have an owner
      effect assigns the specified user as the owner of the resource
    getOwner (resource: Resource): (user: User)
      requires the resource has an owner
      effect returns the owner of the resource
```

2. 

Sync that happens when shortenings are created:

```
sync createAnalytics
  when UrlShortening.register (): (shortUrl)
    Request.shortenUrl(user)
  then
    AnalyticsTracking.create (resource: shortUrl)
    UserOwnership.assignOwner (resource: shortUrl, user)
```

When shortenings are translated to targets:

```
sync trackLookup
  when UrlShortening.lookup (shortUrl)
  then AnalyticsTracking.increment (resource: shortUrl)
```

When a user examines analytics:

```
sync viewAnalytics
  when Request.getAnalytics (shortUrl, user)
    UserOwnership.getOwner (resource: shortUrl): (owner)
    and user is owner
  then AnalyticsTracking.getCount (resource: shortUrl)
```

3. 

- Allowing users to choose their own short URLs: This feature can be realized easily by adding a new synchronization, with no changes required to any of the existing concepts. The UrlShortening.register action was designed with good modularity, as it already accepts a shortUrlSuffix parameter. To implement this, a new sync would be created to handle a user request like Request.createCustomUrl. This sync would directly call UrlShortening.register with the user's chosen suffix, bypassing the NonceGeneration concept entirely for this flow.

- Using the "word as nonce" strategy: This could be realized by modifying the NonceGeneration concept as described in the first section above. No changes to other concepts or synchronizations would be necessary, demonstrating good modularity.

- Including the target URL in analytics: This feature, would be undesirable because it breaks modularity and creates a significant privacy flaw. It would require the AnalyticsTracking concept to also store the targetUrl. This mixes the concern of tracking access counts with the concern of URL mapping, which is properly handled by the UrlShortening concept. In other words it violates the "separation of concerns".

Furthermore, if two different users create short URLs for the same target, this design would aggregate their analytics. This would allow one user to see traffic data generated by another user's link, which is a major privacy and security issue.


- Generate short URLs that are not easily guessed: This would be a change to the implementation of the NonceGeneration.generate action. Instead of a simple counter or words, it could use a cryptographic hash function or a random string generator. This change would be internal to the NonceGeneration concept and would not affect any other part of the design, indicating high modularity.

- Supporting reporting of analytics to creators of short URLs who have not registered as a user: This would be challenging with the current design, which ties ownership to a registered User. A new concept for "unregistered creators" might be needed, perhaps storing a secret key or token provided upon creation. This would require new state and actions, as well as new synchronizations to handle the creation and subsequent analytics requests using this token. The existing design would need to be extended, but the core concepts could likely remain intact.