# Exercise 1

## 1. Invariants

- The count of purchases for a specific item must always be less than or equal to the requested number of that item (in the request).
- Each purchase by a guest/friend must correspond to an existing request for the same item in the same registry.
- The most important invariant is the second one, since the purpose of the registry is for givers to make purchases that the owner actually wants. Without it, purchases could exist that don’t correspond to any request in the registry, which would defeat the correctness and purpose of the whole system.
- The action most affected is **purchase**. It requires that the registry has a request for the item to be purchased with at least the given count. Its effect not only creates the purchase but also updates the request. Therefore, every purchase is linked to an existing request through both the requires and the effect, so the invariant is preserved and the system’s correctness is maintained.


## 2. Fixing an Action

The **removeItem** action is problematic, since it removes the given request from the registry, which could, in turn, leave purchases already made without a request tied to them (and hence break the invariant).

There are a number of ways to fix this:
- Edit the requires: *“Requires a request for this item exists in the registry and no purchases exist for this request.”*
- Alternatively, apply a “hidden” approach (similar to Question 6): instead of deleting a request, mark it as inactive. Current purchases still have a related request, but givers are not able to make new purchases for that specific request.

## 3. Inferring Behavior

A registry can be opened and closed repeatedly.
- `open` requires the registry exists and is not active.
- `close` requires the registry exists and is active.

No other action changes the active status, so these two can be repeated. This is a clever design choice:
- The host might want to temporarily pause the registry to make edits.
- They could also reuse the same registry for future events.
- It is also useful for error recovery (e.g., if the host accidentally opened the registry too early).

## 4. Registry Deletion

No, this does not really matter. The **close** action makes the registry inactive, and **removeItem** removes requests from the registry. Effectively, closing the registry or removing all items has a similar effect to deleting it.

## 5. Queries

- For the registry owner: show what purchases have been made for a given registry (and perhaps by whom).
- For the giver: show what requests are still available, with the remaining quantities, in a given active registry.

## 6. Hiding Purchases

I would edit the set of registrys state and add an `isNotVisible` flag, along with a new action to set this flag.

- **Action:** takes in the registry and the flag.
- **Requires:** registry exists.
- **Effect:** set the flag to true (hidden).

When the flag is true, any owner queries would suppress purchase details (e.g., show only the item but not the giver’s name or quantity).

The flag could be toggled off automatically when the registry closes (by editing the effect of **close**) or via a separate owner-only action.

## 7. Generic Types

This is preferable because identifiers like **SKU codes** ensure uniqueness and stability. In contrast, names or prices can change and may not be unique, which could cause duplication issues.

This design also keeps the concept modular and reusable: it tracks abstract relationships between requests and purchases, rather than embedding specific item details that may not apply in all scenarios.
