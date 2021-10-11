Kindelia: a minimal decentralized computer
==========================================

In 2013, the first smart-contract blockchain, Ethereum, was proposed, extending
Bitcoin with a stateful scripting language that allowed arbitrary financial
transactions to be settled without third parties. Notably, Ethereum's built-in
virtual machine made it a general-purpose computer, even though most of the
protocol's complexity was not required to achieve Turing completeness. Kindelia
is a massive simplification of this concept, trading features for raw
simplicity. Since it does not have a native token, it is not a cryptocurrency
itself, but currencies can be created as smart-contracts. In essence, Kindelia
is a peer-to-peer read-eval-print loop (REPL), making it a minimal decentralized
computer, and a politically neutral decentralized application platform.

Table of Contents
=================

* [Introduction](#introduction)
* [How it works?](#how-it-works)
* [Examples](#examples)
    * [Tokens](#tokens)
    * [Accounts](#accounts)
    * [Transactions](#transactions)
* [How is it so small?](#how-is-it-so-small)
    * [No native accounts](#no-native-accounts)
    * [No native tokens](#no-native-tokens)
    * [A simpler block structure](#a-simpler-block-structure)
    * [A simpler consensus algorithm](#a-simpler-consensus-algorithm)
    * [A simpler virtual machine](#a-simpler-virtual-machine)
* [Specification](#specification)
    * [Types](#types)
      * [Name](#name)
      * [Term](#term)
      * [Operation](#operation)
      * [Type](#type)
      * [Data](#data)
      * [Bond](#bond)
      * [File](#file)
      * [Eval](#eval)
      * [Transaction](#transaction)
      * [Entry](#entry)
      * [World](#world)
    * [Type-Checking](#type-checking)
      * [Var](#var)
      * [Let](#let)
      * [Call](#call)
      * [Create](#create)
      * [Match](#match)
      * [Word](#word)
      * [Compare](#compare)
      * [Operate](#operate)
      * [Set](#set)
      * [Get](#get)
      * [Bind](#bind)
      * [Return](#return)


Introduction
============

Kindelia's design and implementation is, essentially, a massive distillation of
Ethereum, taking away all the complex features that are either historical
artifacts or optimizations, and keeping only the bare minimum required to
establish a decentralized computer and smart-contract platform. For comparison,
our reference implementation requires less than **2000 lines of code**, while
the standard Ethereum full node has [TODO]. Thanks to this simplicity:

## 1. Kindelia is secure

The less code there is, the smaller the surface for attacks and bugs, and the
cheaper it is to fully audit, making Kindelia inherently secure. To further
reinforce this security, its internal scripting language is based on a
low-order, simply-typed calculus that has a measurable cost model (to avoid
spam) and an on-chain type-checker (to prevent reentrancy attacks), making
contracts qualified to formal verification.

## 2. Kindelia is apolitical

Ethereum's complexity makes it politically centralized in the hands of the few
developers that fully understand the protocol. While this is fine for most
users, some will avoid investing in networks that are strongly influenced by
private entities. Kindelia's simplicity make it trivial for companies and
individuals implement their own full nodes, which, in turn, eliminates the class
of "core developers", making it as politically decentralized as possible.

How it works?
=============

Kindelia is, essentially, just a bare-bones functional programming language
running in a peer-to-peer network. Or, in other words, it is a decentralized
read-eval-print loop (REPL). That REPL is split into blocks, which are split
into transactions, which can either declare a new name, a new type, a new
function, or evaluate an expression.

Kindelia nodes keep an ever-growing chain of blocks that are submitted by users,
and sequenced via Nakamoto Consensus (proof-of-work). By evaluating each block
in order, a node can compute the final state of the blockchain, which is just
the set of global names, types and bonds defined on these blocks. For example:

```c
type Nat {
  zero{}
  succ{pred: Nat}
}

bond double(x: Nat) : Nat {
  case x : Nat {
    zero: zero{}
    succ: succ{succ{double(x.pred)}}
  }
}

eval {
  double(succ{succ{succ{zero}}})
} : Nat
```

This is a Kindelia block with 3 transactions. The first declares a type called
"Nat", the second declares bond (function) called "double", and the third
evaluates the expression `double(3)`. The result, `6`, will be logged for
everyone to see. Note that, since Kindelia bonds and evals are type-checked
on-chain, the output will always have the right type. Similarly, cross-bond
communication is always sound.

Of course, the block above doesn't do anything useful. In order to perform
effectful and stateful applications, Kindelia also features files, which are
mutable global variables, and a built-in Effect type, written as `&`, that
allows bonds to write and read from these files. The simplest example is a
counter:

```c
file inc@count : #word = #0

bond inc(): & #word {
  get x     = count
  set count = +(x, #1)
  return #0
}

eval {
  run inc()
  run inc()
  run inc()
  get x = count
  return x
} : & #word
```

The block above declares:

1. `count`: a file, owned by the `inc` bond, that stores a number.

2. `inc`: a bond that increments the `count` file when it is called.

The `eval` block increments the counter 3 types, and outputs `3`.

Notice the return type of `inc` is marked with an `&`: that's because it is an
effectiful bond. A functional programmer may be familiar with it, since it works
exactly like Haskell's IO type. That is, `& #word` in Kindelia is equivalent to
`IO Word64` in Haskell. The `return` primitive is the monadic pure, and `run` is
a short form of the monadic binder:

```c
run x : #word = inc()
```

With types, bonds and files, users can create all sorts of decentralized
applications on Kindelia. The lack of hard-coded currencies and accounts makes
Kindelia flexible and powerful, as it allows users to limit what their accounts
can do, to choose what tokens they'll use to pay miner fees, and to choose their
own authentication methods. While most crypto-currencies would be destroyed by
sufficiently powerful quantum computers, in Kindelia, users can simply opt to
use quantum-resistant signatures.

Since Kindelia blocks are Turing complete, caution is needed to avoid spam such
as endless loops that would overload nodes on the network. As such, blocks are
limited in both space and time by a maximum block size, and a maximum
computation budget. Sadly, assigning gas costs to functional languages is
extremely tricky, because the cost of a β-reduction [may
vary](https://dl.acm.org/doi/10.5555/645420.652523) depending on how it is
implemented. That is why Kindelia's term language is carefully designed to be
strongly confluent. Just like a stack machine, evaluating a Kindelia term has a
clear optimal strategy, allowing us to design a reasonable cost table for it.

Examples
========

## Tokens

A crypto-currency has 3 components: a token, accounts, and transfers. The token
itself can be implemented as a bond that alters a map of balances:

```c
file CatCoin@balances : Map = empty{}

type Command {
  mint{value: #word}
  ...
}

// The CatCoin bond
CatCoin(cmd: Command): #word {
  case cmd {
    mint:
      get balances = balances
      set balances = Map.set(balances, user, cmd.value)
      return #0
    ...
  }
}
```

## Accounts

Similarly, users can create accounts by uploading bonds that they control. For
example:

```c
type Command {
  send_cat_tokens{
    to    : address
    value : #word
    fee   : #word
  }
  ...
}

bond Bob(cmd: Command): & #word {
  case cmd : Command {
    send_cat_tokens:
      if ECDSA.check(hash(cmd), caller)
        run CatCoin.send(cmd.fee, block_miner)
        run CatCoin.send(cmd.amount, cmd.to)
        return #0
    ...
  }
}
```

The bond above can be used by Bob to send cat tokens via ECDSA authentication.
That bond **is** Bob's account. If Bob wished to, he could add more features and
different signature schemes to his account.

## Transactions

Once we have accounts and a currency, sending a token is simply a matter of
including a signed `eval` script in a block:

```c
eval {
  run Bob(send_cat_tokens{@Alice, 50000, 100, "<Bob's sig>"})
  return #0
} : & #word
```

Bob would write this script, sign it, serialize and send to miners. Miners would
be incentived include it in a block, in order to collect fees. Once mined, this
transaction would call Bob's bond, which would check the signature and call the
CatCoin bond, which would bond would update the balance map to send 50000 cat
tokens to Alice, and 100 cat tokens to the block miner.

How is it so small?
===================

In order to become so minimal, Kindelia made several compromises, trading
features, throughput and efficiency for sheer simplicity, security and
stability.  Kindelia aims to be boring like Bitcoin, yet expressive like
Ethereum. Below are some, but not all, of the tradeoff it took:

## 1. No native accounts

Ethereum uses ECDSA for message authentication as a hard-coded algorithm that is
part of the protocol. ECDSA is not only complex, but can be broken by quantum
computers. Kindelia doesn't have a native account system: users can make
accounts by deploying a contracts that they control using their favorite
signature schemes. This makes Kindelia simpler and quantum-resistant.

## 2. No native tokens

Ethereum has a built-in currency, Ether, that is used to pay miner fees.
Kindelia has no native currency, but users can make their own tokens via
smart-contracts, and miner fees can be paid in any of these, with no single one
being special. Transactions don't have a gas limit, but blocks do. As such,
users pay whatever they want, and miners select transactions that fit the
block's gas limit while maximizing their profits. This replicates the fee market
as seen on Ethereum, with more payment options and less built-in complexity.

## 3. A simpler block structure

Ethereum block structure is complex due to both optimizations and historical
artifacts, such as logs and patricia-merkle trees. Kindelia doesn't have layer 1
throughput as a goal, and it doesn't feature light clients. This allows it to
take a minimalist approach, keeping the blockchain structure as simple as
possible: blocks store the previous block hash, a nonce and a list of
transactions, and nothing more.

## 4. A simpler consensus algorithm

Ethereum aims to migrate to a complex Proof-of-Stake consensus algorithm. This
will bring several benefits, such as lower energy consumption and faster
finality times. It also has neat features such as Ethash, for ASIC-resistance,
and GHOST, for mining efficiency. Kindelia drops these features for the sake of
simplicity, featuring just a simple Proof-of-Work onsensus.

## 5. A simpler virtual machine

Instead of a stack-machine with several complex opcodes, Kindelia's built-in
scripting language is a minimal calculus with a very minimal set of operations.
Specifically, it has algebraic datatypes (with pattern-matching), 64-bit
integers (with 8 binary operations and a comparison primitive), recursive
functions, and monadic effects. And that's all.


Specification
=============

Kindelia's virtual machine is based on a core expression language that is
pure, low-order and functional. It features algebraic datatypes, 64-bit unsigned
integers and effects. It doesn't feature high-order functions. That limitation
is necessary for strong confluence (i.e., to have a cost model that doesn't
depend on the evaluation strategy). It does, though, feature branching (via
pattern-matching) and recursive functions, which make it expressive and Turing
complete.

Types
-----

### Name

A name is a string of 6-bit characters on the following alphabet:

```
A B C D E F G H I J K L M N O P
Q R S T U V W X Y Z a b c d e f
g h i j k l m n o p q r s t u v
w x y z 0 1 2 3 4 5 6 7 8 9 . _ 
```

### Term

An expression, or term. Defined by the following abstract syntax tree:

```
Term ::=
  var     (name : Name)
  let     (name : Name) (type : Type  ) (expr : Term  ) (body : Term)
  call    (bond : Name) (vals : [Term])
  create  (data : Name) (vals : [Term])
  match   (name : Name) (data : Name  ) (cses : [Term])
  compare (val0 : Term) (val1 : Term  ) (iflt : Term  ) (ifeq : Term) (ifgt : Term) 
  operate (oper : Oper) (val0 : Term  ) (val0 : Term  )
  set     (file : Name) (expr : Term  ) (body : Term  )
  get     (name : Name) (file : Name  ) (body : Term  )
  bind    (name : Name) (type : Type  ) (expr : Term  ) (body : Term)
  return  (expr : Term)
```

- `var`: a bound variable.

- `let`: a local assignment expression.

- `call`: a call to an external bond.

- `match`: a pattern-match.

- `compare`: a less-equal-greater comparison of words.

- `operate`: a binary operation on words.

- `set`: effect that writes a file.

- `get`: effect that reads a file.

- `bind`: the monadic bind for the Effect type.

- `return`: the monadic return that wraps a pure value.

### Operation

A binary operation on words.

```
Oper ::=
  add
  sub
  mul
  div
  mod
  or
  and
  xor
```

### Type

A type.

```
Type ::=
  word
  data   (name : Name)
  effect (rety : Type)
```


### Data

A global datatype.

```
Data ::=
  new (name : Name)
      (name : [Ctor])

Ctor ::=
  new (name : String)
      (fnam : [String])
      (ftyp : [Type])
```

### Bond

A global function.

```
Bond ::=
  new (name : String)
      (inam : [String])
      (ityp : [Type])
      (otyp : Type)
      (main : Term)
```

### File

A global file.

```
File ::=
  new (name : String)
      (ownr : [String])
      (type : Type)
      (expr : Term)
```

### Eval

A global evaluation.

```
Eval ::=
  new (term : Term)
      (type : Type)
```

### Transaction

A block transaction.

```
Transaction ::=
  new_name (name : Name)
  new_data (data : Data)
  new_bond (bond : Bond)
  new_eval (eval : Eval)
```

### Entry

A global entry.

```
Entry ::=
  data (value : Data)
  bond (value : Bond)
```

### World

The global state.

```
World ::=
  new (names : [Name])
      (entry : Map<Entry>)
```

Type-Checking
-------------

Kindelia terms are statically checked whenever a bond is deployed.

### Var

```
if (name : type) is in context
------------------------------
context |- name : type
```

### Let

```
context             |- expr : A
context, (name : A) |- body : B
--------------------------------------
context |- (let name = expr; body) : B
```

### Call

```
given bond B(x: A, y: B, ...) : T { ... }

context |- x : A
context |- y : B
...
-----------------------------------------
context |- f(x, y, ...) : T
```

### Create

```
given data T { k(x: A, y: B, ...), ... }

context |- x : A
context |- y : B
...
----------------------------------------
context |- k{x, y, ...} : T
```

### Match

```
given data T { k(x: A, y: B, ...), ... }

context                        |- x : T
context, (x : A), (y : B), ... |- r : R
-----------------------------------------
context |- (case x : T { k: r, ... }) : R
```

### Word
    
```
~
---------------------
context |- #n : #word
```

### Compare

```
context |- n : #word
context |- m : #word
context |- l : A
context |- e : A
context |- g : A
-----------------------------------------------------------
context |- (compare n m { _<_: l, _=_: e, _>_: g }) : #word
```

### Operate
  
```
if X is one of: + - * / % | & ^
context |- n : #word
context |- m : #word
-------------------------------
context |- X(n, m) : #word
```

### Set

```
given file a : A

context |- k : A
context |- r : &B
------------------------------
context |- (set a = k; r) : &B
```

### Get

```
given file a : A

context, (x : A) |- r : &B
------------------------------
context |- (get x = a; r) : &B
```

### Bind

```
context          |- e : &A
context, (x : A) |- c : &B
----------------------------------
context |- (run x : A = e; c) : &B
```

### Return

```
context |- t : A
--------------------------
context |- (return t) : &A
```
