## 1.

```plaintext
concept Ownership [User]
purpose record the user that owns a given short URL
principle when a short url is created by a user, ownership is recorded; ownership is deleted when the short url is deleted

state
  a set of Owners with
    a shortUrl String
    an owner User

actions
  createOwnership (shortUrl: String, owner: User)
    requires no ownership exists for shortUrl
    effects create ownership

  getOwner (shortUrl: String): (owner: User)
    requires ownership exists for shortUrl
    effects none

  deleteOwnership (shortUrl: String)
    requires ownership exists for shortUrl
    effects deleted/removes ownership

concept ViewCounts [User]
purpose keep shortened url lookup counts visible only to the owner
principle when a shortened url is looked up, its access count is incremented by 1; only the owner may query/view the counts

state
  a set of Counters with
    an owner User
    a shortUrl String
    a count Number

actions
  createCounter (shortUrl: String, owner: User)
    requires no counter exists for shortUrl
    effects create counter, initialized to count = 0 and owner = owner

  increment (shortUrl: String)
    requires a counter exists for shortUrl
    effects increase count by 1

  getCount (requester: User, shortUrl: String): (count: Number)
    requires a counter exists for shortUrl
             and counter.owner = requester
    effects none

  deleteCounter (shortUrl: String)
    requires a counter exists for shortUrl
    effects deletes the counter
```


## 2.
- _When shortenings are created:_
```plaintext
sync ownAndCount
when
  Request.shortenUrl (requester)
  UrlShortening.register (): (shortUrl)
then
  Ownership.createOwnership (shortUrl, owner: requester)
  ViewCounts.createCounter (shortUrl, owner: requester)
```

- _When shortenings are translated to targets:_
```plaintext
sync incrementOnLookup
when UrlShortening.lookup (shortUrl)
then ViewCounts.increment (shortUrl)
```

- _When a user examines analytics:_
```plaintext
sync viewAnalytics
when Request.getAnalytics (shortUrl, requester)
then ViewCounts.getCount (request, shortUrl): (count)
```

**Notes**:
- these syncs assume our Request has a getAnalytics action like:
 ```plaintext
getAnalytics (shortUrl: String, requester: User)
    effects record the user’s request to view analytics
```
and an additional state:
```plaintext
  a set of AnalyticsRequests with
    a shortUrl String
    a requester User
```

- We also need a deleting sync for completeness:
```plaintext
sync deleteCounterComplete
when UrlShortening.delete (shortUrl)
then Ownership.deleteOwnership (shortUrl)
     ViewCounts.deleteCounter (shortUrl)
```

## 3.

- _Allowing users to choose their own short URLs:_

this could be realized by:

Editing shortenUrl to accept an optional suffix: shortenUrl(targetUrl: String, shortUrlBase: String, optional suffix: String)

Editing the state of Request to:
state
  a set of ShortenRequests with
    a targetUrl String
    a shortUrlBase String
    an optional suffix String

Also, edit the syncs:
  sync register_desired
  when
    Request.shortenUrl (targetUrl, shortUrlBase, suffix)
  then UrlShortening.register (shortUrlSuffix: suffix, shortUrlBase, targetUrl)

Then we adjust the syncs:
If a suffix is provided, call UrlShortening.register(shortUrlSuffix: suffix, shortUrlBase, targetUrl) directly (the register action already ensures uniqueness).

If no suffix is provided, go with the usual path: call NonceGeneration.generate to produce one, then call UrlShortening.register.

**Notes**: Essentially, user-supplied suffixes bypass nonce generation but still go through the same uniqueness check. If no suffix is supplied, the existing random generation flow is unchanged.

- _Using the “word as nonce” strategy to generate more memorable short URLs:_
We can edit the NonceGeneration concept as such:

```plaintext
state
  a set of Contexts with
    a used set of Strings
    a dictionary set of Strings

actions

  setDictionary(context: Context, words: set of Strings)
    effects set dictionary(context) := words / used(context)

  generateWord (context: Context): (nonce: String)
    requires dictionary(context) exists and dictionary(context) is not empty
    effects choose some word that is in the dictionary of the context;
            remove that word from that dictionary of that context;
            add the word to used(context);
```
Also:
```plaintext
  sync generateMemorable
  when Request.shortenUrl (shortUrlBase)
  then NonceGeneration.generateWord (context: shortUrlBase)

  sync registerMemorable
  when
    Request.shortenUrl (targetUrl, shortUrlBase)
    NonceGeneration.generateWord (): (nonce)
  then UrlShortening.register (shortUrlSuffix: nonce, shortUrlBase, targetUrl)
```

**Notes**: with this approach, we keep both approaches (the dictionary one and the traditional random).

- _Including the target URL in analytics, so that lookups of different short URLs can be grouped together when they refer to the same target URL:_

```plaintext
purpose keep shortened url lookup counts visible only to the owner, and also track aggregate counts per target URL
principle when a shortened url is looked up, increment both the short-url count and the target-url aggregate; only the owner may view analytics

state
  a set of Counters with
    an owner User
    a shortUrl String
    a targetUrl String
    a count Number

  a set of TargetCounters with
    a targetUrl String
    a totalCount Number

actions
  createCounter (shortUrl: String, targetUrl: String, owner: User)
    requires no counter exists for shortUrl
    effects create counter with owner = owner, targetUrl = targetUrl, count = 0

  ensureTargetCounter (targetUrl: String)
    effects if no targetCounter exists for targetUrl
              then create a TargetCounter with totalCount = 0
            else do nothing

  incrementTarget (targetUrl: String)
    requires a targetCounter exists for targetUrl
    effects add 1 to targetCounter.totalCount

  getCount (requester: User, shortUrl: String): (count: Number, total: Number)
    requires a counter exists for shortUrl
             and counter.owner = requester
             and a targetCounter exists for counter.targetUrl
    effects none

sync ownAndCount
when
  Request.shortenUrl (targetUrl, requester)
  UrlShortening.register (): (shortUrl)
then
  Ownership.createOwnership (shortUrl, owner: requester)
  ViewCounts.createCounter (shortUrl, targetUrl, owner: requester)
  ViewCounts.ensureTargetCounter (targetUrl)

sync incrementOnLookup
when UrlShortening.lookup (shortUrl): (targetUrl)
then
  ViewCounts.increment (shortUrl)
  ViewCounts.incrementTarget (targetUrl)

sync viewAnalytics
when Request.getAnalytics (shortUrl, requester)
then ViewCounts.getAnalytics (requester, shortUrl): (count, total)
```

**Notes:** a number of changes need ot be made here. I only included the actions/sync/details that needed to be changed. Anyhting not present here means it remains the same.

- _Generate short URLs that are not easily guessed:_

This feature contradicts other features like "word as nonce" as the result will not be memorable. The purpose of the UrlShortening is to: "shorter or more memorable way to link". Having a hard to guess URL defeats that purpose and also it is hard if the URL is short as the possible combinations for a short URL are limited. For a short URL to not be easily guessed, it can just be random, but that is already supported through NonceGeneration technically. However, if we are really particular about implementing this we could just follow some cryptographic approach. This would not really imapct the state/actions, but rather the actual implementation that would need to include more math within the generate action.

- _Supporting reporting of analytics to creators of short URLs who have not registered as user:_

I think this is undesirable, as if the creator does not register, they cannot manage their links. It is also impossible to enfore "owner-only" visibility, as there is no registered owner. Generally, it also weakens security and privacy. Maybe a nice twist on this would be to allow people with tokens. In which case, I would use resource-scoped management tokens (similar to PATs from the previous PSET).

As a brief outline, we could add a short CreatorKeys concept with Keys(shortUrl, secret, active), actions createKey(shortUrl):(secret), validate(shortUrl,secret), revokeKey(shortUrl).

And then, we would also need syncs on:
- UrlShortening.register(): (shortUrl) to CreatorKeys.createKey(shortUrl):(secret). Where the secret would be shown once (like in PATs).
- Request.getAnalyticsWithToken(shortUrl,secret) to CreatorKeys.validate(...) then
ViewCounts.getAnalyticsPublic(shortUrl):(count,total)

And an action (on ViewCounts concept) like:
```plaintext
getAnalyticsPublic (shortUrl: String): (count: Number, total: Number)
  requires a counter exists for shortUrl
           and a targetCounter exists for counter.targetUrl
  effects none
  ```

  **Notes**: the validate action could look like:
  ```plaintext
  validate (shortUrl: String, secret: String)
    requires a key exists for shortUrl
             and key.secret = secret
             and key.active = true
    effects none
  ```
  In general, this approach is very similar to PATs ([see here](../problemset1/problem3.md)). It just requires no registration.
