<pre>
  WIP: WIP-scripting
  Layer: Consensus (hard fork)
  Title: Add scripting to Witnet
  Authors: Tomasz Polaczyk <tomasz@witnet.foundation>
  Discussions-To: `#dev-general` channel on Witnet Community's Discord server
  Status: Draft
  Type: Standards Track
  Created: 2022-08-09
  License: BSD-2-Clause
</pre>


## Abstract

Add scripts to value transfer transactions to allow spending UTXOs with complex conditions.

## Motivation and rationale

Support the following use cases:

* Multisig
* Atomic swap

#### Multisig


#### Atomic swap



## Specification

Scripting language, forth stack machine.

Stack, dont mention alt stack because it is not used.

A script input is an output pointer (UTXO) that needs a `redeem_script` and a `witness_script` to be spendable.

A script output is the same as a normal output: the script address, value, and timelock.

Script addresses are indistinguishable from normal addresses. A script address is calculated as the sha256 hash of the script bytes, truncated to 20 bytes. This is the same hash function used by normal addresses, which hash the public key instead of the script bytes.

#### Supported transactions

Script validation is described in the "Script execution" section. If the script validation is successful, the input can be spent. In case of error, the transaction is marked as invalid and cannot be included in the block.

The following transaction types support scripts:

##### Value transfer

Full support for script inputs and script outputs, this is the intented way to use scripts.

When validating a `ValueTransferTransaction`, for each input, if that input has a `redeem_script`, do a script validation. Otherwise, if the input does not have a `redeem_script`, do a normal signature validation. Transactions can combine both script inputs and normal inputs. Script validation is described in the "Script execution" section.

Interaction with timelock field:

Value transfer outputs have a `time_lock` field which can be used to ensure that the output is not spent until after a certain timestamp. This behavior also applies to scripts, so even if the script is valid, if the value transfer output has a `time_lock` field that is in the future, the transaction must be marked as invalid.

##### Mint

Mint transactions support script outputs. The mint transaction can have up to 2 outputs, and the outputs can have any address, including a script address. The miner must still be a single identity because the block needs a valid secp256k1 signature and proof of eligibility, so multisigs cannot mine blocks. The miner address does not need to be rewarded at all, so the miner can set an external address and send all the minted value there. Mint transactions do not need any new validations because script addresses are equivalent to normal addresses in this case.

TODO: These transactions do not support scripts, assess if some of them should support scripts:

##### Data request

Multiple inputs can be used to pay for the data request. It could support script inputs in that case. The change address can already be a script address, but it must be the same address as the first input, so in practice it cannot be a script address.

##### Commit (collateral)

The inputs used as collateral must all come from the same address. It could support script inputs in that case. The change address can already be a script address, but it must be the same address as the inputs, so in practice it cannot be a script address.

##### Tally (reward payment)

Tally VTTs are created by the miner when creating the tally transaction. The address can already be a script address, but it must be the same address as the collateral input address, which cannot be a script address, so in practice it cannot be a script address.

##### Reveal

Reveal transactions do not use inputs or outputs in any case, so they don't need script support.

TODO: clarify section about why we don't support some transactions


#### Items

A script is a list of items. Each item must be either a value or an operator. During script execution, if a value is encountered then it is pushed to the stack. If an operator is encountered then it is executed. Values always have an explicit type annotation.

#### Types

Unlike in Bitcoin, in Witnet scripts are type-safe. This means it is not possible to apply an operator to a value of a type that is not supported by that operator, any attempt to do so will result in an error and the script execution will halt.

There are 3 types: `Boolean`, `Integer`, and `Bytes`. The `Bytes` type is used to encode addresses, signatures, and hashes.

```rust
pub enum MyValue {
    /// A binary value: either `true` or `false`.
    Boolean(bool),
    /// A signed integer value.
    Integer(i128),
    /// Bytes.
    Bytes(Vec<u8>),
}
```

Integers can take any value in the range of a 128-bit signed integer, from `-170141183460469231731687303715884105728` to `170141183460469231731687303715884105727`, both inclusive.

Bytes can have unlimited size, the are limited only by the block weight limits when the script is encoded.

##### Bytes

Some values are encoded as bytes:

###### PublicKeyHash

Must be encoded as 20 bytes.

###### Signature

Must be encoded as 33 bytes representing the public key, followed by a DER secp256k1 signature encoding.

#### Operators

Summary:

```rust
pub enum MyOperator {
    /// Pop two elements from the stack, push boolean indicating whether they are equal.
    Equal,
    /// Pop bytes from the stack and apply SHA-256 truncated to 160 bits. This is the hash used in Witnet to calculate a PublicKeyHash from a PublicKey.
    Hash160,
    /// Pop bytes from the stack and apply SHA-256.
    Sha256,
    /// Pop PublicKeyHash and Signature from the stack, push boolean indicating whether the signature is valid.
    CheckSig,
    /// Pop integer "n", n PublicKeyHashes, integer "m" and m Signatures. Push boolean indicating whether the signatures are valid.
    CheckMultiSig,
    /// Pop integer "timelock" from the stack, push boolean indicating whether the block timestamp is greater than the timelock.
    CheckTimeLock,
    /// Pop element from the stack and stop script execution if that element is not "true".
    Verify,
    /// Pop boolean from the stack and conditionally execute the next If block.
    If,
    /// Flip execution condition inside an If block.
    Else,
    /// Mark end of If block.
    EndIf,
}
```

Executing any operator with an invalid input type stops script execution with an error.

##### `Equal<T1, T2>(T1, T2) -> Boolean`

Pop two elements from the stack, push boolean indicating whether they are equal.

Both input arguments can have different types, but in that case the comparison will always return false.

##### `Hash160(Bytes) -> Bytes`

Pop bytes from the stack and apply SHA-256 truncated to 160 bits. This is the hash used in Witnet to calculate a PublicKeyHash from a PublicKey.

##### `Sha256(Bytes) -> Bytes`

Pop bytes from the stack and apply SHA-256.

##### `CheckSig(Bytes, Bytes) -> Boolean`

Pop PublicKeyHash and Signature from the stack, push boolean indicating whether the signature is valid.

The PublicKeyHash and the Signature must be encoded as specified in the "Types" section.

CheckSig may return Boolean `false` as the result when signature verification fails, but it can also return an error and stop script execution if the signature is malformed (serialization error).

##### `CheckMultiSig(n: Integer, pkhs: [Bytes; n], m: Integer, signatures: [Bytes; m]) -> Boolean`

Pop integer "n", n PublicKeyHashes, integer "m" and m Signatures. Push boolean indicating whether the signatures are valid.

The first argument "n" denotes how many different Bytes arguments to read next. There is no such thing as an array in the scripting language, the notation `[Bytes; n]` is used to say "pop n elements, must have type Bytes". Same applies to the "m" argument.

The PublicKeyHash and the Signature must be encoded as specified in the "Types" section.

CheckMultiSig may return Boolean `false` as the result when signature verification fails, but it can also return an error and stop script execution if any of the signatures is malformed (serialization error).

##### `CheckTimeLock(Integer) -> Boolean`

Pop integer "timelock" from the stack, push boolean indicating whether the block timestamp is greater than the timelock.

Returns true if `block_timestamp >= timelock`.

##### `Verify(Boolean)`

Pop element from the stack and stop script execution if that element is not "true".

##### `If(Boolean)`

Pop boolean from the stack and conditionally execute the code until a correspoding EndIf instruction.

If the topmost stack element is not a boolean, return an error.

##### `Else`

Flip execution condition inside an If block.

It is an error to use this operator outside an if block.

##### `EndIf`

Mark end of If block.

It is an error to use this operator when there is no corresponding If operator.

#### Script execution

##### Locking script

The first verification when running a script is to check if the script address (also known as locking script, `redeem_script_hash`) matches with the provided `redeem_script`. That check is performed as `hash160(redeem_script_bytes) == redeem_script_hash`:

```rust
// Check locking script
let locking_script = &[
    // Push redeem script as first argument
    Item::Value(MyValue::Bytes(redeem_bytes.to_vec())),
    // Compare hash of redeem script with value of "locking_bytes"
    Item::Operator(MyOperator::Hash160),
    Item::Value(MyValue::Bytes(locking_bytes.to_vec())),
    Item::Operator(MyOperator::Equal),
];
```

If the result of executing the locking script is an error, the script execution stops with that error. If the result of the locking script is anything other than a boolean `true`, the script execution stops with `false`. If after executing the last instruction the number of items left in the stack is greater than one, the script execution stops with `false`.

##### Redeem script

To execute the redeem script, it needs to be prepended with the witness script. The witness script must not contain any operators, it must only contain values.

Script execution is considered successful if the stack ends up containing exactly one item, a boolean "true".

If script execution stops because of an error, such as an operator receving an invalid argument, then the script execution is unsuccessful, regardless of the contents of the stack.

##### Script context

Some script operators need access to external variables such as the block timestamp (for CheckTimeLock operator) or the transaction hash (for signature validation). Therefore, the validity of such scripts depends on the transaction that includes the script input, and also on the block that includes the transaction.

#### Networking changes

Some network messages need to be modified to support the new script transactions.

Scripts are encoded as bytes. The redeem script is a new optional field in the `Input` type:

```rust
pub struct Input {
    pub output_pointer: OutputPointer,
    pub redeem_script: Vec<u8>,
}
```

The redeem script is the part of the script that is used to create a script address.

Additional inputs such as the signature are encoded separately in the `witness` field of the `VTTransaction`.


```rust
message VTTransaction {
    VTTransactionBody body = 1;
    repeated bytes witness = 2;
}
```

The previous version of the protocol had a `signatures` field instead of a witness field. The value of `witness` is deserialized as a script, but depending on the context that script is executed or it is assumed to be a single signature, and then the old signature validation is executed. In the legacy case, the signature is encoded as a script with a single Bytes element, the signature is encoded as described in "Types".

If a transaction has both a `signatures` and a `witness` field, the `witness` field must be ignored. This is for backwards compatibility because old nodes will only see the `signatures` field.

#### Serialization

Scripts need to be encoded as bytes, because they need to be sent over the network, and also because of the locking script check: we need a way to calculate the script hash.

TODO: Currently we use JSON but that needs to change.

## Backwards compatibility

- Implementer: a client that implements or adopts the protocol improvements proposed in this document.
- Non-implementer: a client that fails to or refuses to implement the protocol improvements proposed in this document.


#### Consensus cheat sheet

Upon entry into force of the proposed improvements:

- Blocks and transactions that were considered valid by former versions of the protocol MUST continue to be considered valid.
- Blocks and transactions produced by non-implementers MUST be validated by implementers using the same rules as with those coming from implementers, that is, the new rules.
- Implementers MUST apply the proposed logic when evaluating transactions or computing their own data request eligibility.

As a result:

- Non-implementers MAY NOT get their blocks and transactions accepted by implementers.
- Implementers MAY NOT get their valid blocks accepted by non-implementers.
- Non-implementers MAY consolidate different blocks and superblocks than implementers.

Due to this last point, this MUST be considered a consensus-critical protocol improvement. An adoption plan MUST be proposed, setting an activation date that gives miners enough time in advance to upgrade their existing clients.


### Libraries, clients, and interfaces


## Reference Implementation

A reference implementation for the proposed protocol improvement can be found in the [witnet-rust] repository.


## Adoption Plan

An adoption plan will be proposed upon moving this document from the _Draft_ stage to the _Proposed_ stage.


## Acknowledgements

This proposal has been cooperatively discussed and devised by many individuals from the Witnet development community.

[witnet-rust]: https://github.com/witnet/witnet-rust/
