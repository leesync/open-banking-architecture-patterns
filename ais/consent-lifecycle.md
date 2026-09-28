# Understanding the Account-Information Consent Lifecycle

**Diátaxis type:** Explanation

Account-data access is not a permanent credential. It is a relationship among the user, the regulated third party, the authorisation server and the account provider. The integration must preserve what was authorised, for which accounts, for which purpose and until when.

A useful model separates four concerns:

1. **Consent resource:** the requested permissions, purpose, expiry and access scope.
2. **User authorisation:** the account holder authenticates and approves or rejects the request.
3. **Access credential:** the token or equivalent used to call protected resources.
4. **Local grant record:** the application's auditable view of consent, accounts selected, credential metadata and revocation state.

Consent can end because the user revokes it, the provider rejects it, the authorised period expires, credentials expire without a permitted refresh path, or the product no longer has a lawful or operational reason to continue.

## Why separation matters

A valid access token is not by itself proof that every proposed data use remains authorised. Conversely, an expired token does not necessarily mean the underlying consent is revoked. Model both states and follow the provider's contract.

## Design implications

- Request only permissions needed for the declared user outcome.
- Store consent and credential state separately.
- Make expiry visible before calls begin failing.
- Stop scheduled collection after revocation or expiry.
- Treat reauthorisation as a user journey, not a silent background refresh.
- Retain only the audit evidence and account data allowed by policy and law.

This document explains the model. Use [Implement an account-data access flow](data-access-flow.md) for the task sequence.
