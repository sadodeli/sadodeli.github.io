---
layout: post
title:  Tezos event logging
date:   2022-03-27 15:39:34 +0800
categories: tezos
---

In this blog post, we will discuss a potential extension to the existing Michelson language for the Tezos smart contracts.
This extension will allow smart contracts to deliver event messages to off-chain applications by writing event-like data into the transaction receipt.
The prominent feature of this language extension is that we define a uniform event emitting instruction and a event log entry format that is
sufficiently versatile.
In addition, having event support in Michelson will avoid manual, duplicating and competing definition of critical parts of an event logging system by the users,
so that smart contracts and consumers of events can integrate with each other seamlessly.

# Event logging syntax

## A new instruction

This extension introduces a new Michelson instruction `EMIT`.
It has the following semantics.
    
    EMIT :: string : event : 'C -> 'C
    > EMIT / tag : data : S  =>  S
    
    
The `EMIT` instruction accepts a string `tag` as a tag to the event entry and
additionally a data `data` of type `event` as an attachement.

Notice that there is a type reference `event` in the definition.
This extension will support an optional type declaration, just like `storage` and `parameter`, for `event` at the top level.
In case this declaration is missing, it is assumed to be `never` and, thus, it is impossible to observe any event from this contract ever.
Therefore, an indexing service can directly infer the event type from the contract code.

## Event delivery

With the power of the new instruction, we can now demonstrate
how an event shall be delivered and how indexers can interpret the event data.

Let us take the following contract code as the starting point.

    { parameter unit;
      storage unit;
      event (or (nat %int) (string %str));
      code { DROP;
             PUSH string "right";
             RIGHT nat;
             PUSH string "tag1";
             EMIT;
             
             PUSH nat 2;
             LEFT string;
             PUSH string "tag2";
             EMIT;
             
             UNIT;
             NIL operation;
             PAIR; } }

This code defines the following event type, expressed in CameLigo.
```ocaml
type event = Int of nat | Str of string
```

There are two events emitted in one invocation of the contract.
One is `Str "right"` and the other `Int 2n`.
Upon successful execution of this contract, those two events will be included
into an event log included in the successful transaction operation receipt.

The result of emitting those two events are then visible once the block containing
this successful operation is baked.
Users can easily grab the event log with JSON RPC by initiating a `GET` request directed to
`/chains/main/blocks/<block_id>/operations/<validation_pass>/<offset>`.
A possible response is as follows.

```json
{
  "protocol": "ProtoALphaALphaALphaALphaALphaALphaALphaALphaDdp3zK",
  "chain_id": "NetXdQprcVkpaWU",
  "hash": "opNX59asPwNZGu2kFiHFVJzn7QwtyaExoLtxdZSm3Q3o4jDdSmo",
  "branch": "BLockGenesisGenesisGenesisGenesisGenesisf79b5d1CoW2",
  "contents": [
    {
      "kind": "transaction",
      "source": "tz1KqTpEZ7Yob7QbPE4Hy4Wo8fHG8LhKxZSx",
      "fee": "1000000",
      "counter": "2",
      "gas_limit": "100000",
      "storage_limit": "10000",
      "amount": "0",
      "destination": "KT1Q2cQ97uHhngzjqo1Hk52tDgWJe35MVUEQ",
      "metadata": {
        "balance_updates": [
          {
            "kind": "contract",
            "contract": "tz1KqTpEZ7Yob7QbPE4Hy4Wo8fHG8LhKxZSx",
            "change": "-1000000",
            "origin": "block"
          },
          {
            "kind": "accumulator",
            "category": "block fees",
            "change": "1000000",
            "origin": "block"
          }
        ],
        "operation_result": {
          "status": "applied",
          "storage": {
            "prim": "Unit"
          },
          "consumed_gas": "2064",
          "consumed_milligas": "2063020",
          "storage_size": "122",
          "events": [                                           // <~
            {                                                   // <~
              "emitter": "KT1Q2cQ97uHhngzjqo1Hk52tDgWJe35MVUEQ",// <~
              "payer": "tz1KqTpEZ7Yob7QbPE4Hy4Wo8fHG8LhKxZSx",  // <~
              "tag": "tag1",                                    // <~
              "data": {                                         // <~
                "prim": "Right",                                // <~
                "args": [                                       // <~
                  {                                             // <~
                    "string": "right"                           // <~
                  }                                             // <~
                ]                                               // <~
              }                                                 // <~
            },                                                  // <~
            {                                                   // <~
              "emitter": "KT1Q2cQ97uHhngzjqo1Hk52tDgWJe35MVUEQ",// <~
              "payer": "tz1KqTpEZ7Yob7QbPE4Hy4Wo8fHG8LhKxZSx",  // <~
              "tag": "tag2",                                    // <~
              "data": {                                         // <~
                "prim": "Left",                                 // <~
                "args": [                                       // <~
                  {                                             // <~
                    "int": "2"                                  // <~
                  }                                             // <~
                ]                                               // <~
              }                                                 // <~
            }                                                   // <~
          ]                                                     // <~
        }
      }
    }
  ],
  "signature": "sigsE3tbTcs2DAhJRWFFgtq4FetZ6yQAaBM6rS5Ek186bAzvodWJJ4eEkVHaJUeWUf7DUbwwu6PhSiMob3yWSRXhityAcQnh"
}
```

There are four components in an event log entry. `emitter` is the hash of the contract writing this log entry.
`payer` is the very first initiator of the chain of operations that eventually lead to emitting this event,
most probably a Tezos account calling a smart contract.
The `tag` contains the string that the contract attaches to this entry.
The `data` is a Michelson expression for the event data attachment.

Indexers can also cross-reference the `data` attachment with the `event` type declaration.
This can be done by making a JSON RPC with `/chains/main/blocks/<block_id>/context/contracts/<contract_hash>/script`.
Here is an example.

```json
{
  "code": [
    {
      "prim": "parameter",
      "args": [
        {
          "prim": "unit"
        }
      ]
    },
    {
      "prim": "storage",
      "args": [
        {
          "prim": "unit"
        }
      ]
    },
    {
      "prim": "event",         // <~ event declaration
      "args": [                // <~
        {                      // <~
          "prim": "or",        // <~
          "args": [            // <~
            {                  // <~
              "prim": "nat",   // <~
              "annots": [      // <~
                "%int"         // <~
              ]                // <~
            },                 // <~
            {                  // <~
              "prim": "string",// <~
              "annots": [      // <~
                "%str"         // <~
              ]                // <~
            }                  // <~
          ]                    // <~
        }                      // <~
      ]                        // <~
    },
    {
      "prim": "code",
      "args": [
        [
          {
            "prim": "DROP"
          },
          {
            "prim": "PUSH",
            "args": [
              {
                "prim": "string"
              },
              {
                "string": "right"
              }
            ]
          },
          {
            "prim": "RIGHT",
            "args": [
              {
                "prim": "nat"
              }
            ]
          },
          {
            "prim": "PUSH",
            "args": [
              {
                "prim": "string"
              },
              {
                "string": "tag1"
              }
            ]
          },
          {
            "prim": "EMIT"
          },
          {
            "prim": "PUSH",
            "args": [
              {
                "prim": "nat"
              },
              {
                "int": "2"
              }
            ]
          },
          {
            "prim": "LEFT",
            "args": [
              {
                "prim": "string"
              }
            ]
          },
          {
            "prim": "PUSH",
            "args": [
              {
                "prim": "string"
              },
              {
                "string": "tag2"
              }
            ]
          },
          {
            "prim": "EMIT"
          },
          {
            "prim": "UNIT"
          },
          {
            "prim": "NIL",
            "args": [
              {
                "prim": "operation"
              }
            ]
          },
          {
            "prim": "PAIR"
          }
        ]
      ]
    }
  ],
  "storage": {
    "prim": "Unit"
  }
}
```

# Limitations
There are unfortunately limitations imposed on places where events can be emitted.
Events are not allowed in lambdas and views for a few reasons.

Lambdas can be composed from user inputs, packed, and transmitted between operations.
It is in general not a good idea to allow lambdas from user inputs or other contracts to emit events
as they wish.
It could compromise the functionality or even security of the smart contract.
Further technical difficulty arises when the lambda is getting type-checked,
since the type-checker needs to make sure that the `event` data type needs to match the declared
`event` type of the contract.

Events are currently not considered for support in views because views are supposed to be simple
computation on the immutable contract storage.
There is no serious use case for allowing views to emit event logs.
