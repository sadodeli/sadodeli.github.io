---
layout: post
title:  Tezos event logging, redesigned
date:   2022-06-27 23:00:00 +0800
categories: tezos
---

*This blog post is about an overhaul of the previous design back in March.*

In this blog post, we will discuss a potential extension to the existing Michelson language for the Tezos smart contracts.
This extension will allow smart contracts to deliver event messages to off-chain applications by writing event-like data into the transaction receipt in forms of manager operations.
The prominent feature of this language extension is that we define a uniform event emitting instruction and a event log entry format that is
sufficiently versatile.
In addition, having event support in Michelson will avoid manual, duplicating and competing definition of critical parts of an event logging system by the users,
so that smart contracts and consumers of events can integrate with each other seamlessly.

# How to emit events

Emitting events in a Tezos contract now is intuitive.
A new instruction `EMIT` is introduced for this purpose.
Here is its semantics.

    EMIT %tag 'ty :: 'ty : 'C -> operation : 'C
    > EMIT %tag 'ty / ty : S  =>  operation : S

The `EMIT %tag 'ty` instruction a data of type `'ty` and composes an event operation under a tag `tag` and the supplied data as the event attachment.
Like other operations from `TRANSFER_TOKEN`,
you will need to include the events in a list of operations in the return value of your contract code to effect them.

# Anatomy of events

Events are included in the list of internal operation results in the transaction metadata.
Here is an example of an event emission.

    {
      "protocol": "ProtoALphaALphaALphaALphaALphaALphaALphaALphaDdp3zK",
      "hash": "opNX59asPwNZGu2kFiHFVJzn7QwtyaExoLtxdZSm3Q3o4jDdSmo",
      // ... fields elided for brevity
      "contents": [
        {
          "kind": "transaction",
          // ... fields elided for brevity
          "metadata": {
            // ... fields elided for brevity
            "internal_operation_results": [
              {
                "status": "applied",
                // ... fields elided for brevity
                "source": "KT...",
                "destination": "ev1...",                            // <~
                "parameters": {                                     // <~
                  "entrypoint": "<event_tag...>",                   // <~
                  "value": {                                        // <~
                    "prim": "Right",                                // <~
                    "args": [                                       // <~
                      {                                             // <~
                        "string": "right"                           // <~
                      }                                             // <~
                    ]                                               // <~
                  }                                                 // <~
                }                                                   // <~
              }
            ]
          }
        }
      ]
    }

Notice that on the marked lines, the destination is an address with a prefix `ev1`.
This is a new class of address introduced for external services to identify the events.
The address here is derived from the `'ty` Michelson node as in `EMIT %tag 'ty`.

In addition, the tag will show up as `entrypoint` in the operation `parameters`.
This tag may or may not carry semantics.
However, they are in some scenarios very useful as discussed in the later part of the blog post.

A new Tezos client command is added to help computing an event address from a given Michelson type.
This command can be called as `tezos-client get event address <Michelson Type>`.
For instance, `tezos-client get event address for type 'or (nat %int) (string %str)'` will write out
`Event address: ev12m5E1yW14mc9rsrcdGAWVfDSdmRGuctykrVU55bHZBGv9kmdhW` where the string prefixed with `ev1`
can be used for a trusted identification of the events originating from execution of your contract.

# Locating events

If you are a DApp developer, it is mostly likely that you know the types of event attachment ahead of time.
Therefore, to collect those known events, you may use the pre-computed event addresses to filter out those
events of interest.
For example, ahead of time we may know our contract will emit events with data of type
`pair (nat %amount) (address %owner)` and `or (nat %mint) (nat %burn)`.
In this case, we can compute two event addresses for both types and only collect events with those two destination
from the contract.

## Use tags when typing is unavailable
For indexers that is interested in collecting all events and provide indexing services for applications like
statistics aggregation,
event tags are useful as a hint to indexers as how to interpret the event data.
In this case, collecting events based on tags that an indexer recognizes is enough for this purpose.
In some cases, events could be emitted from lambdas stored in big maps and event tags will be the only efficient way
to filter the events given that inspecting which lambdas exactly emitted them is rather inefficient.

# Example code

So allow us to present you how to emit events with Tezos contracts.
The following code, executed once, will emit two events.

    parameter unit ;
    storage unit ;
    code { DROP ;
           UNIT ;
           PUSH nat 10 ;
           LEFT string ;
           EMIT %event (or (nat %number) (string %words)) ;
           PUSH string "lorem ipsum" ;
           RIGHT nat ;
           EMIT %event (or (nat %number) (string %words)) ;
           NIL operation ;
           SWAP ;
           CONS ;
           SWAP ;
           CONS ;
           PAIR }

The two events will be one with tag `event` and data `Left 10` and one with tag `event` and data `Right "lorem ipsum"`.
The tag will show up as strings `event` in `entrypoint`s and data as `value`s in their respective `parameters`.

# Start testing today with Protocol K

While you have an option to wait for LIGO to support eventing,
you can already start emitting events with
[code embedding](https://ligolang.org/docs/advanced/embedded-michelson) with LIGO.
You can compose a function that takes an event data attachment of type `ty` and write a function
that composes the event internal operation.
The following event in ReasonLIGO shows how this can be done, with `ty` being `nat` and a default entrypoint
if you are not tagging your events.

    let emit_my_event = (n : nat) : operation =>
      [%Michelson ({| {EMIT nat} |} : (nat => operation))](n)

# Final words
We are happy to roll out event logs in Tezos and we believe that with this feature we can greatly improve
the Tezos ecosystem by enabling a variety of applications and modes of interaction between various systems both
on-chain and off-chain.
We hope that you are enjoying this post and get inspired for new ideas powered by this new feature.

Happy hacking!

