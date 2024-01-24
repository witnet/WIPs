<pre>
  WIP: DRAFT-GUIDIAZ-EXTENDED-RADON-ERRORS
  Layer: Consensus (hard fork)
  Title: Extended rules for improved handling of RADON errors in smart contracts.
  Authors: guidiaz
  Discussions-To: `#dev-lounge` channel on Witnet Community's Discord server
  Status: Draft
  Type: Standards Track
  Created: 2024-01-22
  License: BSD-2-Clause?
</pre>

## Abstract

- `*::NoReveals` are renamed to `*::InsufficientQuorum`.
- `*::InsufficientConsensus` are renamed to `*::InsufficientMajority`. 
- Some legacy `RadonErrors` captions are pluralized. 
- `RadError`s other than `InsufficientCommits`, `InsufficientMajority`, `InsufficientQuorum` or `UnhandledIntercept` are to be reported into a small set of new `RadonErrors` values. 
- Additional `RadError` cases are proposed.
- `RadError` cases are now convertible into explanatory `RadonErrors` subcodes.
- One single badly formated Retrieve script will suffice to report a `RadonErrors::MalformedDataRequest` as the Tally result.

## Motivation and rationale

### Refactoring of `*::NoReveals`

While the `RadError::NoReveals` error is generated on the default implementation of a node only if no reveal transactions were received at the deadline epoch on which a data request resolution expires, it would be quite reasonable for custom setups to refrain from evaluating consensus on the resolution of some data request if the number of received reveals at the deadline epoch was just certain percentage below the number of witnesses required by such data request. Therefore, refering **`*::InsufficientQuorum`** would be a more generic description that could be potentially reused on different node setups. 

### Refactoring of `*::InsufficientConsensus`

While there are three different conditions where the protocol determines a "lack of consensus" in the resolution of a data request at its Tally stage, the currently named as `InsufficientConsensus` actually refers to the fact that no majority of values of the same type were revealed. Naming `InsufficientConsensus` one out of three possible reasons for not reaching "consensus" turns out to be a vague description, if not rather misleading. Renaming `*::InsufficientConsensus` to **`*::InsufficientMajority`** states a self-explanatory and clear distinction among the other potential conditions provoking a lack of consensus in the resolution of a data request. 

### Pluralization of some legacy `RadonErrors`

There are some `RadError` cases that can only be thrown at the Retrieve stage of a data request and that may be eventually included within a Tally report, but only if a majority of the data sources turn up triggering the same `RadError` case. 

The same happens with the `RadError::MalformedReveal` and also with the `RadError::EncodeReveal` introduced in [WIP-0026], where a majority of failing revealers (i.e. witnessing nodes) must concur in order for these `RadError`s to finalize into an errored result. 

In coherence with the fact that one single failing entity (data sources during Retrieve stage, or witnessing nodes during Aggregate stage) may not be sufficient for trascending such error codes, their equivalent `RadonErrors` should better be pluralized to:

- `RadonErrors::EncodeReveals`
- `RadonErrors::HttpErrors`
- `RadonErrors::InconsistentSources`
- `RadonErrors::MalformedReveals`
- `RadonErrors::RetrievalsTimeout`

### New reportable error codes

From all possible `RadError`s that can be thrown through the resolution of a data request, only a few have actually transcended to the `RadonReport<RadonTypes>` that gets ultimately serialized into the Tally result of a data request. 

Nevertheless, the kind of a vast majority of error cases have been downcasted to either `RadonErrors::UnhandledIntercept` (i.e. all of the unhandled `RadError`s thrown during any of the Retrieve, Aggregate and Tally stages of data requests), `RadonErrors::TallyExecution` (i.e. the few handled `RadError`s cases that can be potentially thrown during the Tally stage of data requests) or exceptionally to `RadonErrors::Unknown` (i.e. if at least one inconsistent source is detected during Retrieve stage). 

While an error description has been stringified into the final `RadonReport<RadonTypes>` in the first two cases, this has also been proven not to be a very convenient means to infer the actual cause of failing data requests, not to say impractical and economically inviable from the perspective of a smart contract. 

New sets of `RadonErrors` and `RadError`s are proposed as to enable data requesters (including smart contracts) to easily abstract a few general failing reasons:

- **`*::CircumstantialFailure`** ⇒ A majority of data sources could either be temporarily unresponsive or failing to report the requested data.
- **`*::InconsistentSources`** ⇒ At least one data source is inconsistent when queried through multiple transports at once.
- **`*::MalformedDataRequest`** ⇒ Either one or more data sources were badly formated, or so were the Aggregate or the Tally scripts.
- **`*::MalformedResponses`** ⇒ Values returned from a majority of data sources did not match the expected schema.

### Additional `RadError` cases

While `RadError::HttpOther` was thrown whenever there was an issue while composing or executing the HTTP request involved in the retrieval from a data source, the following fine-grained cases will enable the data requester to differentiate formerly unhandable situations:

- **`RadError::InvalidHttpBody`**
- **`RadError::InvalidHttpHeaders`**
- **`RadError::InvalidHttpResponse`**

### All `RadError` cases convertable into explanatory `RadonErrors` subcodes

Even though `RadError`s are to be downcasted into a small selection of first-order error codes, all `RadError` cases are still prone to be referred via a second-order subcode within the serialization of the resulting Tally report. In order to avoid losing track of the originating `RadError`, the conversion into a `RadonErrors::UnhandledIntercept` should never occur from now on, unless some error with no matching `RadError` case was actually produced during the resolution of a data request. 

Also for the sake of conciseness, the downcasting of a `RadError::Subscript` error into a `RadonErrors` value should now be resolved recursively, based on the self-contained `inner` field (i.e. `&Box<RadError>`).

### Forcing data requests with a minority of badly formated retrieval scripts to fail

There are a series of `RadError` cases that provide evidence of at least one Retrieve script being badly formated, but that have been silenced when triggered by just a minority of sources in multi-source data requests. While such partially malformed data requests can be early detected if dry run locally before being broadcasted to the Witnet blockchain, this trial workflow may just not be practical from the perspective of smart contracts that dynamically compose their own Witnet data requests. 

Moreover, whereas some data requests may rely on multiple sources for "redundancy" sake (and so, it would not be a big deal if a minority of sources were badly formated), a vast majority of data requests (e.g. price feeds) do so for the sake of "data composability" (i.e. extracting or deriving an aggregated result from different providers). As to avoid this sort of data requests never coming to provide a complete version of the actual data being requested, a single badly formated Retrieve script must provoke a data request to resolve into a `RadonErrors::MalformedDataRequest` error code. 

> Smart contracts capable of dynamically building multi-source data requests will surely be capable of counter-acting should their data requests fail due to a single source resolving into a `RadonErrors::MalformedDataRequest` error code. 

## Specification

Starting from and including the activation epoch:

**[1]** The following issues will be handled and susceptible to be refered from within an errored result:

- `RadError::InvalidHttpBody`, if some issue arises while composing the body of some HTTP request. 
- `RadError::InvalidHttpHeaders`, if some issue arises while composing the headers of some HTTP request.
- `RadError::InvalidHttpResponse`, if some issue arises while parsing the response to some HTTP request. 

**[2]** All `RadError` cases will have a single first-order `RadonErrors` match according to this mapping:

- **`RadonErrors::CircumstantialFailure`** from:
  - `RadError::ArrayIndexOutOfBounds`
  - `RadError::CircumstantialFailure` *(as for propagating a majority of reveal reports into a Tally report)*
  - `RadError::HttpStatus`
  - `RadError::MapKeyNotFound`
  - `RadError::ModeTie` *(if generated on Aggregate stage)*
  - `RadError::RetrieveTimeout`
  - `RadError::TallyExecution`
- **`RadonErrors:InconsistentSources`** from:
  - `RadError::InconsistentSource`
- **`RadonErrors::InsufficientCommits`** from:
  - `RadError::InsufficientCommits` 
- **`RadonErrors::InsufficientMajority`** from:
  - `RadError::InsufficientMajority`
  - `RadError::ModeTie` *(if generated on Tally stage)*
- **`RadonErrors::InsufficientQuorum`** from:
  - `RadError::InsufficientQuorum`
- **`RadonErrors::MalformedDataRequest`** from:
  - `RadError::ArrayFilterWrongSubscript`
  - `RadError::BadSubscriptFormat`
  - `RadError::HttpOther`
  - `RadError::InvalidHttpBody`
  - `RadError::InvalidHttpHeader`
  - `RadError::InvalidScript`
  - `RadError::MalformedDataRequest` *(as for propagating a majority of retrieve or reveal reports into a Tally report)*
  - `RadError::NoOperatorInCompoundCall`
  - `RadError::NotIntegerOperator`
  - `RadError::NotNaturalOperator`
  - `RadError::NotNaturalOperator`
  - `RadError::ScriptNotArray`
  - `RadError::ScriptTooManyCalls`
  - `RadError::SourceScriptNotArray`
  - `RadError::SourceScriptNotCBOR`
  - `RadError::SourceScriptNotRADON`
  - `RadError::UrlParseError`
  - `RadError::WrongArguments`
  - All `RadError::Unknown*` cases, but *`RadError::Unknown`*.
  - All `RadError::Unsupported*` cases, but *`RadError::UnsupportedSortOp`*.
- **`RadonErrors::MalformedResponses`** from:
  - `RadError::BufferIsNotValue`
  - `RadError::Decode`
  - `RadError::DifferentSizeArrays`
  - `RadError::EmptyArray`
  - `RadError::Encode`
  - `RadError::EncodeReveal`
  - `RadError::Hash`
  - `RadError::InvalidHttpResponse`
  - `RadError::JsonParse`
  - `RadError::MalformedResponses` *(as for propagating a majority of reveal reports into Tally report)*
  - `RadError::MalformedReveal`
  - `RadError::MathDivisionByZero`
  - `RadError::MathOverflow`
  - `RadError::MathUnderflow`
  - `RadError::MismatchingTypes`
  - `RadError::ParseBool`
  - `RadError::ParseFloat`
  - `RadError::ParseInt`
  - `RadError::UnsupportedSortOp`
  - `RadError::XmlParse`
  - `RadError::XmlParseOverflow`
- **`RadonErrors::UnhandledIntercept`** from:
  - All `RadError::DecodeRadonError*` cases.
  - All `RadError::EncodeRadonError*` cases.
  - `RadError::UnhandledIntercept*` *(as for propagating a majority of reveal reports into a Tally report)*
  - `RadError::Unknown`
  
> `RadError::Subscript` will be a special case in which the actual `RadonErrors` value will be recursively determined by trying to find the match of the self-contained inner field (i.e. `&Box<RadError>`). 

**[3]**: Radon reports containing errors of the `RadonErrors::CircumstantialFailure` and `RadonErrors:MalformedResponses` kinds will also serialize a second-order error code (or subcode) as the first item within the error arguments array. This subcode is intended to uniquely specify the originating error that provoked the data request to fail. See below how `RadError` to second-order `RadonErrors` will be mapped from now on. 

Radon reports containing errors of the `RadonErrors::CircumstantialFailure` kind may also serialize additional error arguments:
- `RadError::ArrayIndexOutOfBounds { index: &i32 }` serializes the correspondant error subcode and the `index` value as error arguments. 
- `RadError::CircumstantialFailure { subcode: &u8, args: &Option<Vec<Values>> }` serializes the incoming `subcode` and `args` values as error arguments. 
- `RadError::HttpStatus { status_code: &u16 }` serializes the correspondant error subcode and the `status_code` value as error arguments.
- `RadError::RadError::MapKeyNotFound { key: &String }` serializes the correspondant error subcode and the `key` string as error arguments.
- `RadError::TallyExecution { inner: &Option<Box<RadError>>, message: &Option<String> }` serializes the correspondant error subcode and the `message` value (if any) as error aguments. 

Radon reports containing errors of the `RadonErrors::MalformedDataRequest` kind will first serialize the stage at which the originating error(s) took place as the first item within the error arguments array. If the originating error was singular and triggered during the Aggregate or the Tally stage, the corresponding subcode will be serialized as second error argument. Conversely, if the originating error(s) was/were triggered during the Retrieve stage, an array of tuples will be serialized instead as second error argument. There will be one tuple for every different badly format situation detected on any of the data sources. Every tuple will contain two values: (a) the first value will contain a unique subcode; (b) the second item will contain a subarray with the indexes of the sources that triggered the referred subcode.

**[4]** All `RadError` cases will be univocally convertible into a single `RadonErrors` subcode, except for:
- **`RadError::ModeTie`** will be converted into a *`RadonErrors::ModeTie`* only if produced during Aggregate stage. Conversely, should this error arise during Tally stage, it will be immediately transformed into a `RadError::InsufficientMajority`, for which no subcode actually applies.
- **`RadError::Subscript`** will be converted in whatever `RadonErrors` subcode derives from recursively converting the self-contained `inner` field (i.e. `&Box<RadError>`), until a `RadError` other than `RadError::Subscript` is found. 

While some `RadonErrors` cases will be infered from a single `RadError` case, many others will group together semantically-related `RadError` cases. While a few `RadError` cases will match homonym `RadonErrors` values, the rest will derive in different `RadonErrors` subcodes, namely: 
- **`RadonErrors::ArrayIndexOutOfBounds`** from homonym.
- **`RadonErrors::BufferIsNotValue`** from homonym.
- **`RadonErrors::Decode`** from homonym.
- **`RadonErrors::EmptyArray`** from homonym.
- **`RadonErrors::Encode`** from homonym.
- **`RadonErrors::EncodeReveals`** from:
  - `RadError::EncodeReveal`
  - `RadError::EncodeRadonError**`
- **`RadonErrors::Hash`** from homonym.
- **`RadonErrors::HttpErrors`** from:
  - `RadError::HttpStatus`
  - `RadError::HttpOther`
- **`RadonErrors::MalformedReveals`** from:
  - `RadError::MalformedReveal`
  - `RadError::DecodeRadonError*`
- **`RadonErrors::MapKeyNotFound`** from homonym.
- **`RadonErrors::Math*`** from homonyms.
- **`RadonErrors::MismatchingArrays`** from:
  - `RadError::DifferentSizeArrays`
- **`RadonErrors::ModeTie`** from homonym.
- **`RadonErrors::NonHomogeneousArrays`** from:
  - `RadError::UnsupportedOpNonHomogeneous`
- **`RadonErrors::Parse`** from:
  - `RadError::JsonParse`
  - `RadError::XmlParse`
  - `RadError::ParseBool`
  - `RadError::ParseFloat`
  - `RadError::ParseInt`
- **`RadonErrors:ParseOverflow`** from:
  - `RadError::XmlParseOverflow`
- **`RadonErrors::RequestTooManySources`** from homonym.
- **`RadonErrors::RetrievalsTimeout`** from:
  - `RadError::RetrievalTimeout`
- **`RadonErrors::ScriptTooManyCalls`** from homonym.
- **`RadonErrors::SourceRequestBody`** from:
  - `RadError::InvalidHttpBody`
- **`RadonErrors::SourceRequestHeaders`** from:
  - `RadError::InvalidHttpHeader`
- **`RadonErrors::SourceRequestURL`** from:
  - `RadError::UrlParseError`
- **`RadonErrors::SourceScriptNotArray`** from:
  - `RadError::BadSubscriptFormat`
  - `RadError::InvalidScript`
  - `RadError::ScriptNotArray`
  - `RadError::SourceScriptNotArray `
- **`RadonErrors::SourceScriptNotCBOR`** from homonym.
- **`RadonErrors::SourceScriptNotRADON`** from homonym.
- **`RadonErrors::UnsupportedFilter`** from:
  - `RadError::UnknownFilter`
  - `RadError::UnsupportedFilter`
  - `RadError::UnsupportedFilterInAT`
- **`RadonErrors::UnsupportedHashFunction`** from:
  - `RadError::UnsupportedHashFunction`
- **`RadonErrors::UnsupportedOperator`** from:
  - `RadError::UnknownOperator`
  - `RadError::UnsupportedOperator`
  - `RadError::UnsupportedOperatorInTally`
  - `RadError::NoOperatorInCompoundCall`
  - `RadError::NotIntegerOperator`
  - `RadError::NotNaturalOperator`
- **`RadonErrors::UnsupportedReducer`** from:
  - `RadError::UnknownReducer`
  - `RadError::UnsupportedReducer`
  - `RadError::UnsupportedReducerInAT`
- **`RadonErrors::UnsupportedRequestType`** from:
  - `RadError::UnknownRetrieval`
- **`RadonErrors::WrongArguments`** from:
  - `RadError::MismatchingTypes`
  - `RadError::WrongArguments`
- **`RadonErrors::WrongSubscriptInput`** from:
  - `RadError::ArrayFilterWrongSubscript`
  - `RadError::UnsupportedSortOp`

**[5]** Data requests for which any of the following `RadError` cases are thrown during the Retrieve stage will resolve into a failing report containing **`RadonErrors::MalformedDataRequest`** as first-order error code, no matter if the error case is detected in just one or in multiple data sources:
- `RadError::ArrayFilterWrongSubscript`
- `RadError::BadSubscriptFormat`
- `RadError::HttpOther`
- `RadError::InvalidHttpBody`
- `RadError::InvalidHttpHeader`
- `RadError::InvalidScript`
- `RadError::NoOperatorInCompoundCall`
- `RadError::NotIntegerOperator`
- `RadError::NotNaturalOperator`
- `RadError::NotNaturalOperator`
- `RadError::ScriptNotArray`
- `RadError::ScriptTooManyCalls`
- `RadError::SourceScriptNotArray`
- `RadError::SourceScriptNotCBOR`
- `RadError::SourceScriptNotRADON`
- `RadError::UrlParseError`
- `RadError::WrongArguments`
- All `RadError::Unknown*` cases, but *`RadError::Unknown`*.
- All `RadError::Unsupported*` cases, but *`RadError::UnsupportedSortOp`*.

## Backwards compatibility

**[1]** Renamed `RadonErrors` cases MUST preserve their formerly assigned numeric codes. 

**[2]** Deserialization of Radon reports containing the following `RadonErrors` as first-order error code MUST remain untouched:
- `RadonErrors::ArrayIndexOutOfBounds`
- `RadonErrors::BufferIsNotValue`
- `RadonErrors::EncodeReveals`
- `RadonErrors::HttpErrors` 
- `RadonErrors::InsufficientCommits`
- `RadonErrors::InsufficientQuorum`
- `RadonErrors::InsufficientMajority`
- `RadonErrors::MalformedReveals`
- `RadonErrors::MapKeyNotFound`
- `RadonErrors::Math*`
- `RadonErrors::RequestTooManySources`
- `RadonErrors::RetrievalsTimeout`
- `RadonErrors::ScriptTooManyCalls`
- `RadonErrors::SourceScriptNotCBOR`
- `RadonErrors::SourceScriptNotArray`
- `RadonErrors::SourceScriptNotRADON`
- `RadonErrors::TallyExecution`
- `RadonErrors::UnhandledIntercept`
- `RadonErrors::UnsupportedOperator`

### Consensus cheat sheet

Upon entry into force of the proposed improvements:

- Blocks and transactions that were considered valid by former versions of the protocol MUST continue to be considered 
  valid.
- Blocks and transactions produced by non-implementers MUST be validated by implementers using the same rules as with
  those coming from implementers, that is, the new rules.

As a result:

- Non-implementers MAY NOT get their blocks and transactions accepted by implementers.
- Implementers MAY NOT get their valid blocks accepted by non-implementers.
- Non-implementers MAY consolidate different blocks and superblocks than implementers.

Due to this last point, this MUST be considered a consensus-critical protocol improvement. An adoption plan MUST be
proposed, setting an activation date that gives miners enough time in advance to upgrade their existing clients.

### Libraries, clients, and interfaces

The protocol improvements proposed herein will affect any library or client implementing the core Witnet block chain
protocol, especially block validation rules, transaction validation rules, and superblock consensus. 

## Reference Implementation

A reference implementation for the proposed protocol improvement can be found in the extended-radon-errors branch of the [witnet-rust] repository.

## Adoption Plan

TBD

## Acknowledgements

This proposal has been cooperatively discussed and devised by multiple individuals from the Witnet development community.

[TAPI]: wip-0014.md
[whitepaper]: https://witnet.io/witnet-whitepaper.pdf
[WIP-0026]: wip-0026.md
[witnet-rust]: https://github.com/witnet/witnet-rust

