## 1. Contexts
In the _NonceGeneration_ concept, the contexts exist to avoid any overlap/conflict in the generation of the strings. So, the contexts ensure that when we do URL shortening, we have uniqueness in the string for different bases/apps. For instance, a context could be the base url (like tinyurl.com or bit.ly or some custom base like mit.edu), and then this would be followed by the random string which would be unique within its context. Therefore, we could have tinyurl.com/abcd and bit.ly/abcd and mit.edu/abcd and they would all direct you to the appropriate links, even though the shortened url is the same ("abcd"). In short, contexts ensure separate spaces of uniqueness. This also makes it significantly easier to generate random unique strings, as they only need to be unique within their context (so more possibilities).


## 2. Storing used strings
_NonceGeneration_ must store sets of used strings (within each respective context) to make sure it does not reuse any of them. Therefore, this design choice ensures uniqueness, which is essential for the _generate_ action which must return a string that is not in that set. Otherwise, two or more URLs could map to the same short link.

In an implementation, we could maintain a counter for each context and then incremented it after _generate_ is used. In such a case, the set of used strings in the spec maps to the abstract tracker of how many nonces have been used/are unavailable (which could be represented by a counter). The abstraction function (AF) would then map the counter to the set of all nonces that have been used so far (within of course that context)

In a real implementation, we could have the counter and an encoding function (call it E) that maps a number (call it N) to a string (i.e, that could be digits from 0-9 to letters A-I in the alphabet). And then, every time we use _generate_ it would return E(N), increasing N afterwards. That E(N) would be the nonce.

Then, the AF would be given by: **AF(counter) = {E(n) 0 ≤ n < counter}**


## 3. Words as nonces

- _Advantage:_ Using common dictionary words makes the shortened URLs easier for the user to read them, remember them, type them, and share them.
- _Disadvantage_: On the other hand, because these words are easier to remember/guess, people could just guess the generated URLs, and access websites they were not meant to. This would cause security issues for users who might want their website to be organization-specific for example. This can be fixed by making it password-protected, but this adds an additional layer of complexity that could be avoided with a randomly generated string.

Regardless, to realize this idea, we would need to modify the _NonceGeneration_ concept so that _generate_ gets words from a dictionary rather than just randomly generating strings. We would need some sort of database, or array, to store all those dictonary words and then randomly index in the data structure to obtain one or more of them. In the state, we would need to have the set of available words per context, and the effect of _generate_ action would remove the word(s) that were selected from the set of dictionary words, and add it to the used set. THe purpose would also be changed to "generate unique memorable strings within a context."
