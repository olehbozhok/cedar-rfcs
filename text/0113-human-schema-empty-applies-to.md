<!-- Draft RFC for submission to cedar-policy/rfcs. Rename to text/NNNN-human-schema-empty-applies-to.md once a PR number is assigned. Remember to add this RFC to SUMMARY.md in numerical order. -->

# Accept empty `principal` / `resource` lists in the human-readable schema

## Related issues and PRs

- Prior RFCs:
  - [RFC 0055 — Explicit Unspecified Entities](./0055-remove-unspecified.md). Accepted 2024-05-28, landed in `cedar-policy` v4.0.0. Defines the current semantics this RFC builds on: omitting an `appliesTo` field is a parse error, and **an empty list means the action applies to no entities of that kind**.
- Reference Issues:
  - cedar-policy/cedar#1335 — "improve error message for explicitly empty appliesTo" (closed via #1552). Improved the error wording for the human parser's rejection but did not revisit whether the rejection was correct.
  - cedar-policy/cedar#351 — "Changing semantics of unspecified appliesTo.principalTypes/resourceTypes" (closed, `requires-RFC`). Predates RFC 0055; its concerns were largely resolved by 0055.
- Downstream motivation: Cedarling multi-issuer / partial-authorization flows that call `Authorizer::is_authorized_partial` for actions whose principal type is intentionally absent.
- Implementation PR(s): (to be filled in)

## Timeline

- Started: 2026-04-14
- Accepted: TBD
- Stabilized: TBD

## Summary

RFC 0055 established that in the Cedar schema, an `appliesTo.principal` or `appliesTo.resource` field whose value is an empty list means the action applies to no entities of that kind. The JSON schema parser and the runtime validator already implement this. The human-readable schema (`.cedarschema`) parser does not: it rejects `principal: []` and `resource: []` at parse time with `"'principal' is '[]', which is invalid"`. This RFC proposes to make the human parser accept empty lists, matching the semantics already fixed by RFC 0055 and already implemented in the JSON parser and validator. **This is not a semantic change.** It is an alignment of the human surface syntax with behavior the project has already accepted.

## Basic example

Today (rejected by the human parser, accepted by the JSON parser):

```cedarschema
entity User;
entity Doc;

action View appliesTo {
    principal: [],          // parse error today: "'principal' is '[]', which is invalid"
    resource:  [Doc]
};
```

The JSON equivalent parses cleanly and is already the documented way to express this:

```json
{
  "actions": {
    "View": {
      "appliesTo": {
        "principalTypes": [],
        "resourceTypes": ["Doc"]
      }
    }
  }
}
```

Under this RFC, both forms parse and produce the same internal schema AST. Omitting the `principal:` clause remains a parse error, per RFC 0055.

## Motivation

**1. The human parser is inconsistent with RFC 0055.** RFC 0055 resolved the semantics: empty list means "applies to no entities of that kind." The JSON parser honors this. The validator honors this (`coreschema.rs` produces a clear "no principal types are valid for {action}" message for policies written against such actions). The human parser alone still rejects the syntax. This is an implementation gap relative to an accepted RFC, not a new feature.

**2. JSON ⇄ human round-tripping is broken for this case.** Any schema that legitimately uses `"principalTypes": []` cannot be emitted by the human-schema formatter and cannot be authored by hand in human syntax. Tooling that converts between the two forms fails on otherwise-valid schemas.

**3. There is a real use case: partial authorization with no declared principal type.** Downstream systems evaluate requests via `Authorizer::is_authorized_partial` where the principal is genuinely absent from the schema — for example, Cedarling's multi-issuer flows, where policies gate action on claims rather than on a principal entity type. Today these users must declare a synthetic sentinel entity (`entity NoPrincipal; principal: [NoPrincipal]`) purely to pacify the human parser. The sentinel adds no semantic content, pollutes entity stores, confuses policy readers, and is the exact kind of workaround RFC 0055 was written to remove. RFC 0055 directs users toward application-specific entities *when an entity is conceptually present*; it does not require inventing one when none is.

**4. Prior discussion did not justify the rejection.** #1335 improved the error message only; it did not argue on semantic grounds that empty should be rejected. #351 proposed the stricter direction (disallow empty entirely), predates RFC 0055, and was closed without action. No merged RFC or accepted issue argues that empty lists should fail to parse in the human syntax.

## Detailed design

### Scope

This RFC changes only the human-schema parser (`cedar-policy-core/src/validator/cedar_schema/`). No changes to:

- JSON schema semantics or parsing (already conforming).
- The validator or evaluator (already handles zero-valid-types correctly).
- `Request` construction or authorization APIs.
- RFC 0055's decision that an **absent** `principal:` / `resource:` clause is a parse error.

### Parser change

File: `cedar-policy-core/src/validator/cedar_schema/to_json_schema.rs`.

- The AST-builder path that currently raises `MissingOrEmpty::Empty` for `principal: []` / `resource: []` is removed. An empty list lowers to `principal_types: vec![]` / `resource_types: vec![]` in the internal JSON AST, identical to what `json_schema.rs` produces for the JSON surface syntax.
- The `MissingOrEmpty::Missing` path (absent clause) is **unchanged**; omitting `principal:` or `resource:` continues to be a parse error per RFC 0055.
- The `MissingOrEmpty::Empty` error variant is removed. `MissingOrEmpty` collapses to a single-variant enum (or is replaced by a dedicated `Missing` error type).

The grammar (`grammar.lalrpop`) already parses the empty list form `[]`; no grammar change is required.

### Formatter change

The Cedar schema formatter (`cedar-policy-formatter`) must round-trip `principal: []` and `resource: []` verbatim. Current round-trip tests that assume empty lists are unrepresentable in the human syntax need updating.

### Test changes

- `cedar-policy-core/src/validator/cedar_schema/test.rs`: invert the existing `empty_principal` / `empty_resource` rejection assertions into accept assertions. The `missing_principal` / `missing_resource` rejection assertions remain unchanged.
- Add a round-trip test: JSON with `"principalTypes": []` → human form → JSON preserves the empty list.
- Add an integration test that a policy validating against an action with `principal: []` produces the existing "no principal types are valid for {action}" message, confirming no validator change is needed.

### Semantic model (restated for clarity, not changed)

Per RFC 0055, for each of `principal` and `resource`:

| Surface form        | Meaning                                                   |
| ------------------- | --------------------------------------------------------- |
| clause omitted      | **Parse error.** (RFC 0055)                               |
| `principal: []`     | Action applies to no entities of that kind. (RFC 0055)    |
| `principal: [A, B]` | Action applies to entities of type `A` or `B`.            |

This RFC makes the human parser treat row 2 the same way the JSON parser already does. It does not add or remove any row.

## Drawbacks

- **A mis-author who writes `principal: []` intending "applies to any principal" no longer gets a parse error.** The mistake surfaces at policy validation time instead, with the clear existing message "no principal types are valid for {action}." Linters and IDE tooling may additionally warn on empty lists; that is out of scope for this RFC.
- **Formatter and test churn.** Round-trip and golden-file tests must be updated. Estimated small: single-digit test files.
- **Schema consumers that currently rely on "empty list is a parse error" as a validation step will need to emit a warning instead.** No currently-valid schema changes meaning; no currently-valid schema becomes invalid. In that sense this is not a breaking change for schema authors — only for tools that depend on the rejection.

## Alternatives

1. **Leave as-is; keep the sentinel-entity workaround.** Rejected: the sentinel adds no semantic content, directly contradicts the spirit of RFC 0055's "remove dummy entities" direction, and the inconsistency with the JSON parser remains.

2. **Change the JSON parser to reject empty lists (align downward instead of upward).** Rejected: this would overturn RFC 0055's explicit semantics, require a runtime change, and break every consumer that already uses `"principalTypes": []`.

3. **Introduce a new keyword, e.g. `principal: none` as a sugar for `principal: []`.** Rejected: adds surface-syntax complexity without fixing the underlying inconsistency; the JSON form would still diverge unless a matching JSON change is made.

4. **Accept empty `[]` but also accept an absent clause as equivalent.** Rejected: RFC 0055 deliberately made the absent clause a parse error to eliminate the old "unspecified entity" meaning; reopening it conflicts with a merged RFC.

## Unresolved questions

- Should the formatter canonicalize round-tripped JSON `"principalTypes": []` to explicit `principal: []` (recommended — preserves intent) or to the shortest legal form? Recommendation: explicit `[]`.
- Should the validator optionally emit a lint-level warning when it encounters an empty `principal` / `resource` list, to catch accidental uses? Out of scope here; proposable as a separate schema-lint RFC if desired.
- Survey of sibling `cedar-policy/*` repositories (e.g. `cedar-examples`, the VS Code extension, any third-party formatters) to confirm none depend on the human parser's rejection as a contract. Recommend running this survey during the RFC comment period.
