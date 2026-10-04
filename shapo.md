Exactly. And I think **the descriptor itself should be versioned using the same GUID → SHA mechanism**.

But there's an important distinction: there are really **two kinds of versioning** we need.

### 1. Document instance version

For example:

```text
Customer-123
    ↓
SHA-A
```

then:

```text
Customer-123
    ↓
SHA-B
```

That's the versioning we've already designed.

---

### 2. Document definition version

Now suppose the definition of `Customer` changes.

Version 1:

```text
Customer v1

{
    name,
    address
}
```

Version 2:

```text
Customer v2

{
    first_name,
    last_name,
    address,
    email
}
```

The **descriptor itself** needs a GUID:

```text
CustomerDescriptor
       ↓
      GUID
       ↓
     SHA-D1
```

Then:

```text
CustomerDescriptor
       ↓
     SHA-D2
```

So we get:

```text
Customer
  │
  ├── instances
  │      ├── SHA-101
  │      ├── SHA-102
  │      └── SHA-103
  │
  └── definition
         ├── SHA-D1   ← schema v1
         └── SHA-D2   ← schema v2
```

That is much more powerful than simply putting:

```text
version = 2
```

inside the document.

The **version is itself an immutable object**.

---

## I'd make the descriptor look roughly like this

```text
DocumentDescriptor
{
  guid

    version

  parent_sha

    fields
    references
    constraints

    migration
}
```

For example:

```text
CustomerDescriptor v2
{
  guid: CustomerDescriptorGUID

    version: 2

  parent_sha:
        SHA-D1

    fields:
        first_name: String
        last_name:  String
        address:    Address
        email:      String
}
```

Now you have an explicit evolution chain:

```text
D1
 ↓
D2
 ↓
D3
 ↓
D4
```

And because each descriptor is immutable, you can always reconstruct exactly what a document meant at a particular point in time.

---

# But I would go one step further

I wouldn't make `version` the thing that identifies the definition.

The SHA already does that.

So:

```text
CustomerDescriptor v2
```

really means:

```text
CustomerDescriptor
    SHA = ABC123...
```

and `version=2` is human/application metadata.

This gives us:

```text
GUID → SHA
```

for **both** documents and their definitions.

That's very elegant.

---

## Then an actual document can say exactly which definition it uses

For example:

```text
Customer-123
    ↓
SHA-C123
```

and:

```text
SHA-C123
{
    descriptor: SHA-D2

    first_name: "John"
    last_name:  "Smith"
    ...
}
```

So we can answer:

> What did this document look like?

by looking at `SHA-C123`.

And:

> What did the definition mean when this document was created?

by following:

```text
SHA-C123
   ↓
SHA-D2
```

That's **historically reproducible**.

---

# This also solves schema migration

Suppose:

```text
D1 → D2
```

and an old object is:

```text
Customer SHA-A
    descriptor = D1
```

We don't mutate it.

Instead:

```text
SHA-A
  │
  │ migrate
  ▼
SHA-B
  descriptor = D2
```

So migration is itself just another immutable transformation:

```text
D1 object
   ↓
migration
   ↓
D2 object
```

And the old version remains valid forever.

---

## This gives us a really nice invariant

> **A SHA identifies immutable content; the document's version context makes that content interpretable.**

Conceptually:

```text
Document version context
    -> Content SHA    -> Immutable content
    -> Descriptor SHA -> Immutable descriptor
```

Distinct documents may share the same content SHA. Everything required to
interpret a document version can be reconstructed from its content and version
context, not from the content SHA alone. The storage layout of that context
remains an open question.

That is, I think, a much stronger foundation than having a mutable "schema registry" somewhere outside the object store.

The store provides a native `DocumentDescriptor` type, retrieved through the same
typed `get` API as other types. Application definitions are documents created
using this native type. Its bootstrap implementation remains to be specified.

---

## Document API Decisions

`create_document` generates the document GUID internally. The examples use
`guid` for a descriptor's embedded identity and `parent_sha` for a content's
parent link. These are content conventions, not mandatory fields in every value.

`set(value)` creates a new immutable version and returns a document view fixed
to that version. Existing views retain their GUID, SHA, and value. All versions
of the same document share its GUID.

An ordinary local write also advances the local head. Comparing the head with
the receiver's base version and updating it must be atomic. A stale receiver
cannot silently overwrite a newer head: this produces a local conflict. This
check is local to one store, not a global lock across stores.

The API separates a document's identity from the version held by its caller:

```javascript
const customer_v2 = customer_v1.set(value);

customer_v1.sha();       // SHA of v1, unchanged by the write.
customer_v2.data();      // Typed data represented by the v2 view.
customer_v2.rawData();   // Exact stored string; bytes in the future.
customer_v2.previous();  // Parent view when History is supported; null at the root.
customer_v1.head();      // New view of the currently known local head.
```

`get<T>(guid)` on a store or transaction retrieves a typed document view;
`get<T>()` on a store retrieves the type definition, as in the existing examples.
Documents expose `data()` and `rawData()` instead of `get()`.
`data()` returns data decoded according to the exact definition associated with
the held version. Changing the returned data cannot implicitly modify the document.
`rawData()` returns the exact stored content without decoding or transformation:
a string for now, bytes in the future. Both methods read the held version, not
the current head. `set(value)` still accepts a raw string in the current examples.
The decoding contract and its errors remain to be specified; JSON is only an
application format. The boundary between decoded data and descriptor metadata
also remains open.

`sha()` returns the content SHA of the held version. For now, the examples hash
the entire supplied string exposed by `rawData()`, without excluding fields.
`head()` does not mutate the receiver and does not establish a globally accepted
head. A version without locally available content may still be known by reference.

The exact descriptor version associated with a historical state must remain
recoverable. Whether its reference belongs inside the hashed value or outside it
is still under discussion; the earlier diagrams illustrate an embedded layout.

## Conflict Resolution Decisions

Ordinary conflicts are handled by a versioned policy declared by the type.
Transaction-level policies for invariants spanning multiple documents or types
will be addressed separately; individual resolutions must not bypass them.

For an exceptional case, `set` may receive a local resolver while still returning
the document directly:

```javascript
const document = previous.set(value, {
    resolveConflict: async (conflict, resolveDefault) => {
        if (!needsSpecialHandling(conflict)) {
            return resolveDefault();
        }

        return chooseValue(
            conflict.commonAncestor,
            conflict.local,
            conflict.incoming
        );
    }
});
```

The callback is attached to this write in the local store. It receives version
views and returns a proposed value; the engine validates and records the
resolution. The callback is not transmitted to peers: the resulting resolution
is announced instead. Original versions remain unchanged, and arrival order
alone does not select a globally accepted winner.

`needsSpecialHandling` and `chooseValue` are application placeholders, not store
operations. Failure to produce a valid resolution leaves the conflict open.
Policy execution, representation, and distributed convergence remain to be defined.

## Open Design Tasks

The numbers below refer to the specification review, not implementation order.

- [ ] Points 2 and 5: Preserve the exact definition version used by each document
  state, and defer where that association is stored and how it affects hashing.
  Publishing definition v2 does not automatically migrate documents using v1.
  A type-definition change
  should ideally leave the content SHA unchanged when the value's bytes do not
  change. The starting store model has two maps: document GUID -> value reference
  and content SHA -> content. Determine whether descriptor context fits in those
  maps, requires additional metadata, or needs a selective hashing mechanism.
  No layout or hashing exclusion is selected; the examples still hash the entire
  supplied string for now.
- [ ] Point 9: Rework optional traits, including History and embedded GUID support;
  define their declarations and contracts without imposing a content format.
- [ ] Point 10: Design type indexes and search on a store, separately from GUIDs.
- [ ] Point 11: Specify migrations declared in the target definition, referencing
  the previous definition and a migration function; execution remains deferred.
- [ ] Point 12: Specify internal synchronization: reference announcements before
  content, provenance versus availability, missing history, and conflict propagation.
- [ ] Point 14: Define document-bound await milestones, durability guarantees,
  peer acknowledgments, errors, timeouts, and cancellation. Historical confirmations
  do not promise permanent availability or freedom from future conflicts.
- [ ] Point 16: Clarify how a resolution references multiple parents and how
  History navigation works in that case; the current examples do not settle this.
- [ ] Point 17: Clarify what `resolved` confirms for a version and how to obtain
  its exact resolution without assuming that a subsequent `head()` still points to it.
- [ ] Design data calculations and changes triggered by ordinary `set()`, including
  writes outside transactions. When defining that capability, revisit transactions:
  completion waits, subsequent reads, automatic tracking of triggered writes,
  atomicity, and failure handling. No timing contract or `settle()` API is selected.