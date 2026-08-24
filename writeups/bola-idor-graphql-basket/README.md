# BOLA / IDOR — Unauthorized Manipulation of Another User's Basket via GraphQL

> **Platform:** Bugcrowd
> **Category:** Broken Access Control (BAC) → Insecure Direct Object References (IDOR)
> **VRT:** Modify/View Sensitive Information — Complex Object Identifiers (GUID/UUID)
> **Status:** Duplicate

## Overview

A **Broken Object Level Authorization (BOLA / IDOR)** vulnerability was identified within the shared THG e-commerce infrastructure.

An authenticated attacker (**User A**) could manipulate and query the shopping basket belonging to another authenticated user (**User B**) by supplying User B's `basketId` in the variables of a GraphQL mutation.

The backend correctly validated that the attacker possessed a valid authenticated session, but failed to verify whether the authenticated user was authorized to access or modify the supplied `basketId`.

The issue affected basket operations including:

* Updating item quantities;
* Removing/deleting items from another user's basket.

Additionally, the GraphQL response exposed adjacent checkout and metadata structures. During testing, the response included the `paymentMethods` property, indicating additional potential exposure within the returned object structure.

---

## Affected Scope

The issue was reported across the following basket endpoints:

* `https://www.lookfantastic.com/basket/`
* `https://www.myprotein.com/basket/`
* `https://www.myvitamins.com/basket/`
* `https://www.mankind.co.uk/basket/`
* `https://www.matalan.co.uk/basket/`
* `https://arrowfilms.com/basket/`

The affected functionality relied on the shared GraphQL infrastructure.

---

## Technical Details

The application uses a GraphQL API to perform operations against shopping baskets.

A basket is identified through a `basketId`, which is supplied as part of the GraphQL mutation variables.

The authorization model correctly established that the request originated from an authenticated user, but the backend did not adequately validate the relationship between:

```text
Authenticated User
        │
        │ authorization check
        ▼
Requested basketId
```

As a result, a valid authenticated session belonging to **User A** could be used together with a `basketId` belonging to **User B**.

The vulnerable condition can be represented as:

```text
User A
  │
  │ Valid authenticated session
  ▼
GraphQL API
  │
  │ basketId = User B
  ▼
User B's Basket
```

The server processed the operation instead of rejecting the request because the requested basket did not belong to the authenticated user.

---

## Proof of Concept

The vulnerability can be reproduced using two authenticated users.

### User A — Attacker

User A authenticates normally and captures a legitimate basket mutation using an intercepting proxy such as Burp Suite or Caido.

The legitimate request contains User A's `basketId`.

### User B — Victim

User B has a separate shopping basket identified by a different `basketId`.

### Manipulation

The `basketId` supplied in the GraphQL mutation variables is replaced with the identifier belonging to User B.

Conceptually:

```diff
{
  "variables": {
-   "basketId": "USER_A_BASKET_ID"
+   "basketId": "USER_B_BASKET_ID"
  }
}
```

The request continues to use **User A's authenticated session**.

No authentication bypass is required.

The relevant security boundary is therefore the authorization check between the authenticated session and the requested basket object.

---

## Vulnerable Operations

The issue was observed in multiple basket operations.

### UpdateBasketQuantity

An authenticated User A could supply User B's `basketId` to the `UpdateBasketQuantity` mutation.

The operation was processed against User B's basket instead of being rejected.

This allowed the quantity of items in another user's basket to be modified.

### Item Removal / Deletion

The same authorization issue was also present in item removal/deletion functionality.

By supplying another user's `basketId`, an authenticated attacker could remove items from the target basket.

This demonstrated that the issue was not limited to a single operation.

---

## Response Behavior

After submitting the manipulated GraphQL mutation, the server returned:

```http
HTTP/1.1 200 OK
```

and processed the operation against the supplied basket.

The response also contained the updated basket payload and adjacent checkout-related structures.

One of the observed properties was:

```text
paymentMethods
```

This was documented in the original report as a potential additional exposure within the GraphQL response structure.

No claim is made here that complete payment credentials were exposed; the relevant observation was the presence of the `paymentMethods` property in the returned structure.

---

## Impact

The vulnerability allowed an authenticated attacker to interact with another user's shopping basket by providing the victim's `basketId`.

The demonstrated impact included:

* Modifying item quantities;
* Removing items from another user's basket;
* Manipulating the state of another user's shopping cart;
* Returning checkout-related structures in the GraphQL response.

Because the issue affected authorization at the object level, possession of a valid authenticated session was sufficient to reach the vulnerable functionality; an authentication bypass was not required.

---

## Why This Is BOLA / IDOR

The vulnerability exists because the application trusted a client-supplied object identifier without adequately verifying whether the authenticated user was authorized to operate on that object.

The security model should enforce:

```text
User A + User A basket → ALLOW

User A + User B basket → DENY
```

Instead, the observed behavior allowed:

```text
User A + User B basket → OPERATION PROCESSED
```

This represents a **Broken Object Level Authorization** condition and falls within the **Insecure Direct Object Reference (IDOR)** vulnerability class.

---

## Recommended Remediation

The server should perform an authorization check for every basket operation.

Before processing a mutation, the backend should verify that the requested `basketId` belongs to, or is otherwise authorized for, the authenticated user.

Conceptually:

```text
Authenticated User
        │
        ▼
Retrieve requested basket
        │
        ▼
Verify ownership / authorization
        │
    ┌───┴───┐
    │       │
Authorized  Unauthorized
    │       │
    ▼       ▼
Process    Reject
```

Authorization must be enforced **server-side** and independently for every operation that accesses or modifies basket objects.

Client-side restrictions should not be relied upon as an access-control mechanism.

---

## Responsible Disclosure

The vulnerability was responsibly reported through Bugcrowd.

During triage, the submission was classified under:

**Broken Access Control (BAC) → Insecure Direct Object References (IDOR) → Modify/View Sensitive Information (Complex Object Identifiers GUID/UUID).**

The submission was subsequently marked as **Duplicate** because another researcher had previously reported the same issue.

The Bugcrowd response associated the duplicate with:

**Horizon — Update Another User's Basket Quantity**

The referenced submission had been submitted on **16 July 2024**.

The duplicate classification does not change the technical behavior observed during my own research; it indicates that the underlying issue had already been reported by another researcher.

---

## PoC Video

The following video demonstrates the proof of concept described in this report, including the authenticated session, manipulation of the `basketId`, execution of the GraphQL operation, and the resulting behavior.

https://github.com/user-attachments/assets/b89371ab-549f-4752-9168-7dd2d93d8d2b

---

## Conclusion

This research demonstrated a **Broken Object Level Authorization (BOLA / IDOR)** vulnerability within GraphQL-based basket functionality.

The application authenticated the requesting user but failed to enforce authorization over the basket object referenced by the client-supplied `basketId`.

As a result, an authenticated user could manipulate another user's basket through operations such as quantity modification and item deletion.

The vulnerability was responsibly disclosed through Bugcrowd and was ultimately classified as a **Duplicate** after the platform determined that the same issue had previously been reported.

This case provided practical experience in:

* GraphQL security testing;
* Broken Access Control analysis;
* BOLA / IDOR identification;
* Object-level authorization testing;
* HTTP request interception and manipulation;
* API vulnerability research;
* Impact assessment;
* Responsible vulnerability disclosure.
