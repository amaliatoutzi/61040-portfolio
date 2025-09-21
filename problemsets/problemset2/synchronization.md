## 1. Partial matching
In the first sync (_generate_), the when clause only includes shortUrlBase since only this argument is needed to make a new and unique suffix. The base acts as the context in this case. The targetUrl is not needed since we generate a suffix, so we don't include it as an argument.

On the other hand, in the second sync (_register_), includes shortUrlBase and targetUrl. This is because the _then_ clause, needs both to map them. In this case, the _then_, maps the targetUrl to the shortened link.

So, the purpose of the two syncs is different. In the first, we simply generate, which requires the shortUrlBase, vs. in the second sync, we map the shortened link (base + nonce) to the target link. We notice that the _when_ clause must have exactly the variables that the respective _then_ needs.

## 2. Omitting names

Names can be omitted when the argument name and the bound variable name are the same, since there is no ambiguity. Nonetheless, if a variable maps to more than one argument, or if there are several variables whose roles could be confused, then the names should be written explicitly. This way, the spec makes it clear how the same value flows into multiple arguments.

## 3. Inclusion of request

It depends on who initiates the action. For instance, the first two syncs include the _Request_ action they are triggered by the user (when the user submits a long URL/base and asks for it to be shortened). On the other hand, setting the expiry (third sync) is triggered by the completion of the _UrlShortening.register_ action (not by the user directly). The wishes of the user are reflected through the request, so when the user does not trigger diretly an action, the request does not need to exist. Also, this ensures we keep concerns separated. We also maitnain the invariant that there is always an expiration date since as long as the registration has been succesful, there will always be an expiration (regardless of the details of the request itself).

## 4. Fixed domain

To implement this I would change the _generate_ sync by editing the then to something fixed: NonceGeneration.generate(context: "bit.ly"). In the same sync, I would also change the when to remove the argument as it is no longer needed:   when Request.shortenUrl (). Also, the _register_ sync would also change its then to then UrlShortening.register(shortUrlSuffix: nonce, shortUrlBase: "bit.ly", targetUrl). I would also change its when to:     Request.shortenUrl (targetUrl), given that the base is fixed now.

This approach ensures we do not make any changes to the actual implementations (we only change the arguments). Alternatively, we could remove any mention of "bit.ly" completely, and hard code it within the implementation itself. (so basically we keep a single specific context: "bit.ly").

## 5. Adding a sync

```plaintext
sync completeExpire
when ExpiringResource.expireResource(): (resource)
then UrlShortening.delete(shortUrl: resource)
```
