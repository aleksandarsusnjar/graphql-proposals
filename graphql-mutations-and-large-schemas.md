# GraphQL Mutations & Large Schemas

## Important GraphQL background bits

1. Queries fields can be fetched independently and concurrently, in any order 
   the API server/implementor chooses.
   Mutation "fields", operations really, already have to be executed serially.

2. Mutations are said and thought of doing their work (serially),
   then starting to act as fetches/queries (in any order and/or concurrently).

3. Mutations and queries otherwise share the same definition/schema abilities
   and are, in the end, just "fields".
   
4. Many interpret (1)-(3) as "mutations can only be declared at the root level."
   Reasons are multiple:
   
    a) Mutations are understood to be followed by fetches (2) so despite (3),
       one cannot nest mutations as that would mean that there are mutations
       inside queries.
   
    b) Many think of including multiple interdependent mutations but due to (1)
       the order of nested mutations would not be understood as deterministic.
       In fact, with some servers it probably would not be. Since the operations
       cannot be ordered in this way, it seems that this is a no-go.
   
5. Others (including me) see (4) as a big issue. We have hundreds of types in
   large schemas, interrelated. Products are flexible and, often, even user
   extensible. The number of possible operations is huge. Including every 
   operation at the root level, even if it were possible, would present a 
   huge namespace challenge (possibly hundreds of thousands of operations) 
   and would render the on-boarding with the API extremely difficult. 
   This is not about "microservices". We sell on-premise hardware boxes exposing
   these APIs. They are what they are, and they need to serve their function.
   
6. One way to reduce/address (5) is to break up the operations into smaller
   pieces and allow the client to invoke multiple. This is often not possible,
   especially during creation - sufficient, yet complex and large, input
   is required. Breaking up would literally mean reshaping the product 
   around the (in)ability of the API tech.
   
7. Another way to reduce/address (5) is to use verbose (deep, nested) input
   structures that can supply all needed data. However, this is problematic
   in large and deep schemas and especially when there are polymorphic inputs.
   At the same time some functionality is lost - input values do not have
   argument, require explicit lists for repetitions, not sure about redirection.
   They don't "return" anything either, so any interest in their outcomes
   must be restated/duplicated in the "fetch" section - inside `{...}`.

## Relevant links to existing texts

I found the following online:

### Content about/with nested mutations

* https://blog.logrocket.com/supporting-opt-in-nested-mutations-in-graphql/
* https://graphql-api.com/guides/schema/using-nested-mutations/
* https://github.com/oney/graphql-mutate
* [Issue 252 - Proposal: Serial fields (nested mutations)](https://github.com/graphql/graphql-spec/issues/252)
* https://graphql-api.com/blog/released-graphql-api-v07-with-mutations-and-nested-mutations/
* https://github.com/Shopify/graphql-design-tutorial/issues/2
* https://graphql-by-pop.com/docs/operational/embeddable-fields.html

### Descriptions of the order/concurrency issue

* https://www.freecodecamp.org/news/beware-of-graphql-nested-mutations-9cdb84e062b5/

### Uses of the approach (7) - nested inputs

* https://hasura.io/docs/latest/graphql/core/databases/postgres/mutations/multiple-mutations.html
* https://medium.com/hackernoon/graphql-nested-mutations-with-apollo-small-fatigue-max-gain-1020f627ea2e
* https://platform.ontotext.com/tutorials/graphql-mutation.html
* https://graphql-rules.com/rules/mutation-namespaces

## My thoughts

### Important general background note

Most people find it easier to think of "changes" in a sequential fashion.
This, however, is not a requirement for decades. Databases, for example,
can and do perform many update/insert/delete transactions concurrently,
via independent connections. Any modern server (such as a database server) 
exposing any API (including GraphQL) has to be ready for concurrently
arriving requests.

Data integrity is guarded by various means including locks of different
kinds and synchronization. For example, if the two following requests
come in (from whatever origin(s)):

1. Deposit $100 to the chequing account of customer 1.
2. Close the chequing account of customer 2.

... the server first needs to check whether those customers/accounts
are the same (which may end in a temporary table or database lock or
otherwise, for example) but, from that point on, if it finds that they
are separate, it may handle the two requests concurrently, in other
cases it may need to delay one until the other is processed
(sufficiently to release any locks).

Even if those two actions arrive within the same request, from the same
origin the database implementation can choose to behave exactly the same
and attempt to concurrently process parts it finds to have no dependency
between them. For example:

1. Deposit $100 to the chequing account of customer 1.
2. Deposit $200 to the savings account of customer 1 (the same customer).

In here the sequential execution may have to be enforced by the server 
until it realizes that this is about two different accounts
(barring any complex cross-account logic on the way out), but the two
account updates are independent and could be performed concurrently.

Whether the operations happen concurrently or not, we live in the world
where operations get automatically reordered anyway within some layer 
of abstraction between the layer(s) above and the database, for example.
Validation/constraints wise, a single operation may not be able to maintain
data integrity and we may need two or more to occur before any validation
is executed, which is why we have "deferred constraints" at the database
level.

With locking (of various types) and deferred constraints we have the means
to confirm clean executions (and commit transactions) and/or to detect
collisions (and fail, roll transactions back, which can then be retried).

In the end, it becomes apparent that sequential and/or parallel execution
is something that the server could and/or may do but it is the client that
is responsible for ultimately choosing whether, based on what it wants to
accomplish, the sequential execution is required (it can never demand
parallel execution, only hint it). There are many existing ways this is
done, depending on language/protocol/means, in order from the simplest 
(but also least capable) to the most complex (and most capable)
implementation:

1. Separately execute requests in a sequential fashion.
2. Flag a batch of requests as sequential.
3. Extension of (2): denote any (sub-)section of the batch as sequential.
4. Extension of (3): denote any (sub-)section of the batch as sequential or
   parallel, allow nesting of one type inside another.
5. Declare that some action(s) is/are dependent on completion of
   (an)other(s) (primitive workflow).
6. Extension of (5): declare that the input(s) of some action(s) is/are
   sourced from the output(s) of (an)other(s) (workflow with data flow).

Note that none of these are exclusive and can easily evolve from (1) to (6).

### Single complex (deep, nested) mutation != batch of mutations

Mutations and queries are technically the same except with respect to intent
and notes on serialization / order / concurrency that seems to be the greatest
detractor of nested mutations.

However, this is actually not an issue. I'll explain.

They may seem related but they are not - there is an important difference.

Think of a typical textbook case of a transaction - moving funds from, say,
chequing account to a savings one. In some oversimplified pseudo-GraphQL
this could look like (please excuse the lack of use newlines as usual; 
I omitted some for brevity and better document layout):

```GraphQL
mutation {
    updateChequing(input: {customerId: "123", delta: -100}) { balance }
    updateSavings(input: {customerId: "123", delta: 100}) { balance }
}
```

(Please ignore infinite reasons why such an API would be bad and note that 
this is just an illustration.)

In whatever order these operations execute the outcome will be the same.
We don't have to rely on the fact that mutations are meant to be serialized.
We can treat the whole thing as a transaction, or request one to be using
whatever means we have available (if that isn't a default). All that matters
is that the total sum of deltas/transfers is 0. In this case the chequing
account's balance would go down by 100 and savings up by 100, regardless
of order.

In my case I treat the entire request as a single transaction by default. 
Since this is so, then the following would be equally safe:

```GraphQL
mutation {
    updateCustomer(input: {id: "123"}) {
        updateChequing(input: {delta: -100}) { balance }
        updateSavings(input: {delta: 100}) { balance }
    }
}
```

In fact, in reality, due to how many things work, the following would
likely succeed on initially empty accounts:

```GraphQL
mutation {
    updateCustomer(input: {id: "123"}) {
        # remember - before this the balances of both accounts are 0
        updateChequing(input: {delta: -100}) { balance }
        withdrawFromSavings(input: {amount: -50}) { balance }
        updateSavings(input: {delta: 100}) { balance }
        depositToChequing(input: {amount: 200}) { balance }
    }
}
```

... because all these operations are viewed together, not in any
particular order. Joint criteria is that the sum of transfers 
must be 0 (zero) and that the ending balances of all accounts are 
non-negative (or greater than they were, if negatives are allowed).
In the above example the outcome is always:

1. Chequing account's balance goes up by 200 + (-100) = 100.
2. Savings account's balance goes up by 100 + (-50) = 50.
3. The bank has 200 + (-50) = 150 more cache.

### When the order is needed (different case)

Sure, there can be operations that demand order. But there are options
for those. For example, they can be broken up and brought back to the 
serialized root or, if that is not sufficient, actually broken up
into separate (HTTP or other) requests. Examples of both, lazily reusing
the same case as above as if it were sensitive to order:

**Internal break-up**
```GraphQL
mutation {
    updateCustomer(input: {id: "123"}) {
        depositToChequing(input: {amount: 200}) { balance }
    }
    updateCustomer(input: {id: "123"}) {
        updateChequing(input: {delta: -100}) { balance }
        updateSavings(input: {delta: 100}) { balance }
    }
    updateCustomer(input: {id: "123"}) {
        withdrawFromSavings(input: {amount: -50}) { balance }
    }
}
```

**Complete break-up**
```GraphQL
mutation {
    updateCustomer(input: {id: "123"}) {
        depositToChequing(input: {amount: 200}) { balance }
    }
}
mutation {
    updateCustomer(input: {id: "123"}) {
        updateChequing(input: {delta: -100}) { balance }
        updateSavings(input: {delta: 100}) { balance }
    }
}
mutation {
    updateCustomer(input: {id: "123"}) {
        withdrawFromSavings(input: {amount: -50}) { balance }
    }
}
```

### When the order is not a (primary) concern

What many of us are challenged by is the namespace, readability, expressivity
issue and that does not have to be a problem at all because for most of these
cases there is no (or needs to be no) serialization issue. There is an implicit
order often overlooked - a nested field cannot be processed before the outer
(wrapper) one has a chance to do so.

In typical forum terms, while it is true that multiple replies to the same
message can be posted concurrently, a message needs to be processed before
it can be replied to - the "depth-bound serialization" is alive and well.

The outer/wrapper field can do what they are designed to do and supply the
context, both naming and otherwise to nested fields that can continue making
(related) changes. In the above examples, account updates will inevitably
occur _after_ containing customer updates get their chance.

**This is sufficient**. Data extraction / response forming can happen in any
order too. There is only one more thing that _needs_ to happen in predictable
order: the containing/wrapper fields must be able to perform validation and,
possibly, finalization, after all nested fields have been completely 
processed (in any order) and before the transaction is committed.

**Another example**

Consider the following:

(Please excuse the omission of variable definitions for brevity)

```GraphQL
mutation {
    A(input: $inA) {
        A1(input: $inA1) {
            A1x(input: $inA1x) { something }
            A1y(input: $inA1y) { something }
        }
        A2(input: $inA2) {
            A2x(input: $inA2x) { something }
            A2y(input: $inA2y) { something }
        }
    }
    B(input: $inB) {
        B1(input: $inB1) {
            B1x(input: $inB1x) { something }
            B1y(input: $inB1y) { something }
        }
        B2(input: $inB2) {
            B2x(input: $inB2x) { something }
            B2y(input: $inB2y) { something }
        }
    }
}
```

##### Notice:
1. The "cores of" `A` and `B` mutations will themselves be done in a serial fashion.
2. Today there are no guarantees about `B*` happening in any order
   relative to their `A` counterparts or themselves other than what is noted below.
   That is perfectly acceptable for some use cases. Others can break stuff up.
3. `A*` are already guaranteed to happen after `A` and `B*` are guaranteed to
   happen after `B`.
4. `A1*` are already guaranteed to happen after `A1` and `B1*` are guaranteed to
   happen after `B1`.
5. `A2*` are already guaranteed to happen after `A2` and `B2*` are guaranteed to
   happen after `B2`.    
6. A server _can be implemented_ if developers so choose to give `A1` a second
   "look" at the state of affairs after all `A1*` are processed. Equivalent is
   true for `A2` with `A2*`, `B1` with `B1*` and `B2` with `B2*`. Then
   we go one level up and the same is true: `A` can get a second look after
   `A*` are done and `B` after `B*` are done.
7. The entire transaction can be validated at the end.

In essence, this could be an example of one "convoluted" order:

0. Transaction begins.
1. `A` is processed.
2. `b` is processed.
3. `A1`, `A2`, `B1` and `B2` are processed concurrently and/or in any order.
4. `A1x`, `A1y`, `A2x`, `A2y`, `B1x`, `B1y`, `B2x` and `B2y`
   are processed concurrently and/or in any order.
5. `A1`, `A2`, `B1` and `B2` perform finalization concurrently and/or in any order,
   as soon as there nested mutations are complete.
6. `A` and `b` perform finalization.
7. Transaction is committed if there are no violations or errors.

Again, for constructing complex mutations this is fine in a very large number
of cases. We are not talking about cases where the path taken matters, only
the destination.

## No specification change needed?

This does not require the spec to change, it is not going against it:

1. By default, with no info to the contrary, the clients cannot assume any
   order and have to play along with that. They will have to apply one of the
   mutation break-up strategies.
   
2. A particular single server can _choose_ to execute any and all mutations
   in whatever order (or concurrently), as it sees fit. It could use internal
   synchronization to bring some order to otherwise concurrent but
   inter-dependent execution too.
   
3. A server may document it behaviour in standard GraphQL documentation and
   using directives. It could indicate that order is preserved and that
   clients can take advantage of this, if they know about it. If they do not,
   they will revert to (1), which remains a safe option. Thus, even though
   a standard/official directive is preferred, it is not required. 
   
4. Federation middleware can, but does not need to support this. It is 
   already responsible for delegating relevant chunks of requests to 
   relevant servers. To them the nested mutations will look like unbreakable
   chunks (because they will be entirely handled by a single server) 
   and they will themselves serialize the root-level ones.
   
5. While this does affect `@defer` and `@stream`, it only affects the 
   back-end implementation, not the GraphQL spec or the client. Specifically,
   it is up to the server to decide what it takes for any particular
   fragment or field to be ready. Query and mutation fields that do not
   require (any further) handling "on the way out" can be sent sooner
   than those that do (from the same scope) and are implicitly "deferred" 
   that way. The only question that remains is whether the server is 
   also able to send anything but a monolithic response.
   
## Benefits

1. Namespacing - root only contains entry points, nothing more.
   
2. Context - mutations don't have to require what is already available in 
   the context in their input.

3. Documentation - it is immediately apparent which mutations are relevant 
   in which contexts and they are easier to find due to both (1) and (2).
   
4. Mutation simplicity  - mutations can be kept small and simple. 
   Complex mutations are composed of relevant, meaningful elemental 
   mutations. 
   
5. Consistency is a lot easier to accomplish than with a huge number 
   of complex mutations.
   
6. Shorter mutation names.

7. Due to all of the above, GraphQL appears more welcoming to products
   with complex/large schemas, though evolving supporting libraries always
   helps.

## "Seed ideas" for further brainstorming

> **IMPORTANT**
> 
> Please keep in mind that these are all "adjustable" and only
> show some possible evolutions of the standard.
> While I did spend some not insignificant time on this, 
> as I believe this is an important issue, note that I am 
> explicitly open to improvements and iterative approaches.
> I urge you, however, to not dismiss the need to act.
> The challenge here may not seem significant to those who
> didn't face it, likely precisely because they did not face it.
> Others may have shied away from GraphQL without voicing 
> their concerns.

### No technical spec change but add guidance

As noted above, the spec does not have to technically change as long as the
involved are aware of implications of this and the appropriate / safe behaviour.

This would mean:

1. The clients continue can assume that root-level mutations are sequential,
   somewhat unfortunately but needed for compatibility.
2. The clients continue to NOT assume that anything nested is sequential and
   must split requests if any particular ordering is important.
3. Server implementors are advised that any operations nested within the roots
   should be accomplished in a manner that is independent of the order and may
   fail ambiguous requests (i.e. result in error) in which different orders may
   yield different outcomes. They are also advised that concurrent execution is
   not a requirement, just an option.

We (c/w)ould also introduce official guidance on how nested mutations are best
exposed and in which scenarios they make sense.

### Introduce a schema directive stating server's approach

This is essentially, as I understand, the concept of "serial types". It would
provide official means to state the difference between `Query` and `Mutation`
themselves too.

I advise against this, though, as this impacts the client behaviour in the inverse
fashion of what GraphQL does with fields. It is the client that needs to request
something, not the server telling the client to adjust the behaviour. Think 
of what would happen if the server changed the type from being "serial" to
not being serial, for whatever reason. The clients would suddenly have to start
breaking up their requests to ensure any order they feel is important. 
If the client drives this from the beginning, this is not a concern.
Hence the next idea.

### Server abilities directive + client order request directive

It is generally impossible to request operations to occur in parallel. This may
only be "permitted", not required. As such, we recognize two cases:

1. The operations can happen in any order.
2. The operations must happen in certain/specified order.

(1) is the default for `Query` and (2) is the default for `Mutation`.
(2) is a harder/additional requirement compared to (1). A server may
(or not) be able to honour it. Today it has to honor (2) for root level
mutations only and clients cannot assume that it is for anything else.

Instead of using a schema directive to state that a type is "serial",
a server may use a schema directive to indicate that it
_could honour the specified order (2) if client requests so_. It does
not matter, actually, whether it may be doing so anyway or why it may
not be doing it (maybe it does this concurrently, maybe it just parses
operations into an unordered collection). 

Clients that are aware of this could request this behaviour with a 
matching client-supplied directive inside the request.

There are multiple sub-variations of this approach. The schema-side
server capability directive and the client-supplied non-schema directive
can be stated on either:

a) (output) type only, starting with `Mutation`

b) field only - see [Issue 252 - Proposal: Serial fields (nested mutations)](https://github.com/graphql/graphql-spec/issues/252)

c) both of the above - types and fields

I actually think/feel that (b) provides most of the power with least
of the changes (and could be specified on the `Schema.mutation` field
too). This also opens up the possibility of ideas mentioned later
in a cleaner way than (a) would.

As this is about server abilities we may introduce a single root 
directive to hold this and any future server abilities or we may use 
a dedicated directive. For example:

```GraphQL
type Mutation {
    foo(input: FooInput): FooOutput @orderable
    foo(input: FooInput): FooOutput @sequentialAble
    foo(input: FooInput): FooOutput @seqAble
    foo(input: FooInput): FooOutput @canHonourOrder
   ...
}
```
or:
```GraphQL
    foo(input: FooInput): FooOutput @serverAbilities(canHonourOrder: true)
```
(names are, of course, adjustable)

The client would have to, then, request a particular order by  using
a related directive such as `@orderedExecution`, `@seq` or similar.
If omitted, the client accepts that any order is fine.

Given this, a server desiring more could actually implements fields
literally named, for example, `seq @orderable` and `par` (not orderable)
allowing the clients to choose that way / mix and match for old and new:

```GraphQL
mutation {
    foo {
        seq {
            ...
            par {
                ...
                seq {
                   ...
                }
                ...
            }
            ...
            par {...}
            ...
        }
        ...
    }
}
```

... but there are at least a few issues with this:

1. This "feels" as if it does affect the scope/namespace/nesting of the
   response. GraphQL's nesting is already more than significant so, I think,
   this should be avoided.
2. I've cheated and omitted the `@orderedExecution` client directive on
   `seq` fields, which makes this invalid and takes us to the next idea.
3. Client directives are easily ignored. A server that could do concurrent
   execution may end up doing it even when sequential execution is requested.

### Introduce explicit syntax for "order is important" blocks

This is essentially making the `seq` and `par` example from the idea above
legal and automatically apply/imply the `@orderedExecution` directive.
We can talk about what that syntax could be. I thought of it a bit and
came up with the following:

1. It would be nice if it does not cause another level of nesting in
   the output or in the field paths from the root.
2. As always, it is best if it is simple and somehow familiar.

One possible way may be to use the square brackets `[...]` instead of the 
braces `{...}` in the "order is important requests. Square brackets are
used in many languages to indicate (ordered) arrays, including 
GraphQL (lists), JavaScript and JSON, Java, Python. 
Essentially the following:

```GraphQL
mutation {
    foo(input: $fooInput) [ # <-- notice the [
        mustComeFirst(input: $barInput) {...}
        mustComeSecond(input: $barInput) {...}
    ] # <-- notice the ]
}
```
would be equivalent to:
```GraphQL
mutation {
    foo(input: $fooInput) @orderedExecution {
        mustComeFirst(input: $barInput) {...}
        mustComeSecond(input: $barInput) {...}
    }
}
```

This is safe:

1. Legacy clients do not need to know to issue requests like this
   and will not be confused by the new schema directives they don't 
   care about.
2. Legacy clients will never issue these requests with order-is-important
   expressed any way.
3. New clients, aware of this feature, will only request this from
   servers that indicate they are capable of this.
4. The servers can, at any point, decide to add this capability.
5. Servers that do not implement (or signal support for this)
   will fail the requests as they can't parse them. This properly
   prevents arbitrary order of execution when specific order is
   legally requested.
7. While removing the capability entirely could yield incompatibility,
   it is safe to keep it advertised (to maintain compatibility) while
   actually continuing to do things in sequential order only.
   Yes, if the server is reimplemented from scratch and can't parse
   honour the ordering request because it always does things out of
   order, that would be incompatible, but they would have a clear path
   to relatively easily maintain said compatibility with all clients.
   
Counter-arguments:

1. Takes the syntax away from something else that may need it,
   though at a scope appeares safe and well-defined/isolated.
2. Points at an inconsistency introduced before. Technically the
   `Schema.mutation` requests would need to use `[...]` to truly
   request order. This behaviour, unfortunately, seems to have
   to remain being a discrepancy - it is the inherited state.
3. A way to indicate that the server could perform things 
   concurrently, if it is able to do so, on `Schema.mutation`
   requires a separate, different solution.
   Perhaps `{{` - double brace or another, opposite, directive
   as it may be a rare case (and would only be needed on
   `Schema.mutation`). Such a directive could be entirely ignored
   by all but the most capable servers. Perhaps `@anyOrder`?
4. Array syntax associates with arrays that have no name-value
   pairs, just values. Specifying aliases for field requests 
   may look out of place. Merging requests may also look out of
   place. In that sense something like `{!...}` and `{~...}` or
   `!{...}` and `~{...}` (bring your own idea) may feel 
   better.
   
   
### Forget about sequences, express dependencies

I'll start with an example as I think it might be the fastest way to 
describe:

```GraphQL
mutation {
   foo(input: $fooInput)  {
      nested(input: $nestedInput) @dependsOn(operation = $mutation.blah) {
         ...
      }
   }
   bar(input: $fooInput) {...}
   blah: other(input: $fooInput) @dependsOn(operation = "bar") {...}
}
```
Is it clear?

If present, these dependencies may override the other ordering logic.
Yes, the server support for this would need to be signalled somehow
and clients should only use this if this is available. As this
includes the syntactic changes it may need to be scoped to the entire
schema but this would complicate federation a lot, as federation 
middleware would have to support it for all back-ends or not at all
and would have to know how to split requests for back-ends that
can't guarantee order. That, in turn, may become an issue due to
possible data inconsistency between multiple transaction (which may
not be the case otherwise with sufficient transaction isolation).

Leaving support per-field would, however, require that this is
supported on all field within any referenced path too.  

It should be easy to federate too.

The operation path may be:

1. Dedicated syntax would ensure that servers will fail (as they should)
   if this is requested but they don't support it. Here I used a
   variable-like syntax but with dots in the name, presently not allowed
   in variable names. Alternatively something else can be used, maybe
   `$$` or following the `$` sign immediatelly with a dot `.`. 
   While other representations can be accomplished much more easily,
   such as using strings or lists/arrays, there would be a risk of servers
   that don't support this feature failing to fail. Dedicated syntax
   also allows for better client-side validation.
2. This may evolve into supporting relative or scoped paths.
   For example, the `mutation.` prefix may become unneeded.

### Why stop there? Add data flow.

```GraphQL
mutation {
   foo(input: $fooInput)  {
      nested(input: {
         arg1: $mutation.blah.id # Take the id from the blah output
      }) {
         ...
      }
   }
   bar(input: $barInput) {...}
   blah: other(input: $blahInput) @dependsOn(operation = $mutation.bar) {
       id
   }
}
```

> Note: resulting order: bar -> blah -> foo.

While this does start to look like complex orchestration, it can help
bring together what would otherwise have to be separate requests and 
is rather simple. It would also reduce the need to have mutations that
need to expose complex query inputs to find what they should be applied
to, thus reducing the number of operations. Example:

```GraphQL
mutation {
   joe: findUser(where: {name: { eq: "Joe"}}) {
      emailAddress
   }
   sendEmail(input: {
      to: $joe.emailAddress
      subject: "Hi Joe!"
      body: "Are we doing this?"
   }) {
      emailId
   }
}
```

This has similar server capability declaration considerations
as the `@dependsOn`.

Thi, however, blurs the distinction between the input and output
types (which must be reusable as inputs). Scalars can do 
this now but complex types would have to as well.

Additional challenge is dealing with a multiplicity of query outputs
from which a field needs to be extracted. It is important
to note that these collections can only be considered "complete" 
when all the fields up/out to the nearest common scope with the 
referrer are complete. Otherwise the dependents would be executed 
with partial results only or individually, multiple times.
Multiple executions may be needed, but each needs to operate on 
exactly the matching set/list of values. That needs a workflow-type
discussion that can be organized  separately. Please note that, 
at this point this "idea" is just a seed and an indication of 
future possibilities.

### Anonymously nested queries

A further progression of the previous idea allows direct nesting
of queries as input parameters instead of referencing them by path.
For example:

```GraphQL
mutation {
   sendEmail(input: {
      to: findUser(where: {name: { eq: "Joe"}})*.emailAddress
      subject: "Hi Joes! Yes, all of you!"
      body: "Are we doing this?"
   }) {
      emailId
   }
}
```

Here I used a Groovy-like `*.` spread / flattening operator as well
to indicate simple mapping/extraction of individual fields of a
collection of owner objects into a collection of those field values.
For non-List fields the spread isn't needed but it is otherwise 
essentially all but required. This issue, with somewhat different
proposal is listed in 
[Issue 174 - \[RFC\] Flat chain syntax](https://github.com/graphql/graphql-spec/issues/174).

### Please note

Other than the second "idea" (a field variation on serial types, existing proposal),
the rest fall within a single, compatible evolutionary line. In that
sense the "serial type" looks odd and out of place, may be a potential
dead end (although there could certainly be evolutions from there too).

In that sense, the "lesser" ideas seem safe(r) as there is a known 
possible and desirable progression from them. Directives are generally
safe too. The syntactic extensions / sugar is likely the most
"sensitive" change and would need to be additionally discussed.

## Conundrums and adjacent issues

### Should scalars remain in input arguments or be nested?
While, in this model, it makes perfect sense to use nested mutations
for complex changes resulting in relations (edges) changes, for example,
the scalar mutations can either remain in `input` argument(s) or
have their own fields.... which would need their own arguments for input.
Having a common/standard way would be preferential.

### "I don't know yet" Void type   
Some mutations have to "stretch" to "invent" any sort of output.
This is not directly related to the subject of this document but gets
more pronounced due to more mutations being elementary. Problem with
having to invent output is the risk of being wrong. Using dedicated
complex types requires them to have meaningful fields. Cheating with
scalars (including custom scalars, such as `Void`) would cause 
compatibility to be broken if this ever needs to change due to the
requirement that the selection set is not empty.
Why is that a requirement?

This is not to say that some sort of output is not useful or that it
will never be. It is just that, on the way to back-end implementation,
GraphQL keeps forcing the decision to be made at some point about scalar
output and forces the clients to request what they may not care about
yet. A true `Void` type here would imply indicate "we didn't get to
decide on this yet and will return nothing *for now*, you don't have
to request anything and nothing will appear in the output, not even 
`null`".
   
### Polymorphism, ids and naming
Supporting polymoprhism in GraphQL often requires multiplication of
operations, essentially to include the specific type name within the
operation name. One alternative approach are to embed types within ids
but this works only when the id is, in fact, known. Yet another 
alternative is to use a composite/pair of type (enum, to allow validation)
and raw id but that doesn't allow one to state relation between those
type enums and the other inputs or outputs. GraphQL could benefit
from evolving this, maybe even supporting some sort of generics - i.e.
one could have a generic `search` query that, among other inputs
takes a type and states that it will yield instance of that type
(or subtype).
