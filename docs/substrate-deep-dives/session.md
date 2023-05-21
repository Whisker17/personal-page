---
id: session
title: Session Pallet
tags:
  - Substrate
  - Deep Dives
---

As we know, Account Abstraction(AA) has been a hot topic in Ethereum recently, with the intention of further lowering the onboarding barrier for users, and an essential feature of AA is the introduction of Session keys.

Session keys are helpful in that they free users from the need to constantly sign their on-chain operations, a poor user experience that has been criticized for a long time. It is impossible for users to accept exposing their private keys to risk in order to gain some convenience, and Session keys solve this problem.

## Introduction to Session Keys

Firstly, let's introduce the meaning of `Session` in Substrate; `Session` in Substrate refers to **a period of time with a constant set of validators** and we are restricted to only changing the set of validators when the Session is changed.

Session keys are a set of hotkeys held by the validators for performing on-chain operations, and we change Session keys as the set of validators changes. In effect, Session keys are several keys kept together which provide the various signing functions required by network authorities/validators to perform their duties. Each session key plays a specific role in consensus or security in general. All session keys derive their authority from a session certificate

* A Session Certificate is a message containing all session keys signed in tandem, signed by the Controller.

## Code Implementation

For the Session Pallet, our most basic functions are **the configuration of Session keys** and **the Rotation of Sessions**.

### Session keys configuration

**Session keys are set with** `set_keys` **and are not used in the next Session but after the next Session.** They are stored in `NextKeys`, which is a mapping between the caller's `ValidatorId` and the provided session key. `set_keys` allows the user to set their session key before being selected as a validator. This is a public call because it uses `ensure_signed`, which checks whether the origin is a signed account. Therefore, the origin account ID stored in `NextKeys` is not necessarily associated with the block author or validator. **Once the account balance is zero, the account's session key will be deleted.**

### Session control

Instead of specifying a limit on the length of a session, we would normally determine the start of a new session through the implementation of `ShouldEndSession`. The pallet provides a `PeriodicSessions` structure for simple periodic sessions. The following is an implementation of the `ShouldEndSession` trait for the `PeriodicSessions` structure:

```rust
fn should_end_session(now: BlockNumber) -> bool {
    let offset = Offset::get();
    now >= offset && ((now - offset) % Period::get()).is_zero()
}
```

Use `should_end_session` to determine if you want to stop the session in the current block

* Make sure the current block number is greater than `Offset` (`Offset` is the length set for the first session).

* also sets `Period` as the length of the session

At the same time, we also implement the

* `average_session_length` to get the length of the period

* `estimate_current_session_progress`, which calculates the progress of the session in which the current block number is located

* `estimate_next_session_rotation`, which calculates the block number of the next session to be executed

We need to be clear that if we want to open a new Session, we can choose to provide a new set of validators

* Even if our validators don't change, we must provide a new set of validators if there are any potential changes in economic conditions, such as stake weights.

* We need to be explicit about changes to the consensus engine for the set of validators to ensure that errors are correctly located

We also need to have implemented the `SessionHandler` trait-related function

* `on_genesis_session`, where we will provide a collection of validators for the genesis session

* `on_new_session`, to make it clear that the first time we call `on_new_session`, the set of validators needs to be the same as the genesis session, and note that we can also call this function before initializing the pallet

* `on_before_session_ending`, a reminder before the end of the session, we can still modify the set of validators as it is not yet finished

* `on_disabled`, a validator is disabled until a new session starts.

### Session Rotation process

At the beginning of each block, the `on_initialize` function queries the provided implementation of `ShouldEndSession`. If the session is to end, the newly activated validator ID and session key are retrieved from storage and passed to `SessionHandler`. The set of validators provided by `SessionManager::new_session` and the corresponding Session keys (which may have been registered during the previous session via `set_keys`) are written to storage and they will wait for a session before being passed to `SessionHandler`.

* The following is the most critical implementation

  * `on_initialize`, Here we will call `should_end_session` to determine if a new session can be started, and if so we will perform the following on the appropriate block number via `rotate_session`

  * `set_keys`, allows an account to set its own session key before becoming a validator, which is not valid until the next session

  * `purge_keys`, removes all session keys from the function caller, not valid until the next session

  * `rotate_session`, will register a new set of validators and session keys

        1. first call `on_before_session_ending` and `end_session` to notify handlers that the session is ending

        2. get the session keys and validators in the queue

        3. increment `session_index` and call `start_session`

        4. get the next set of validators

        5. insert the next session keys into the queue

        6. call `on_new_session` to tell everyone that new session keys are available

  * `disable_index`, disable index for the ith validator.

    * We are doing the insertion in an ordered way, so we can optimize the lookup with a binary search.

  * `upgrade_keys`, update the type of keys, need to be used with care, only in the `on_runtime_upgrade` block.

## Summary

If we could have a UI/UX friendly session keys setup platform, we could greatly lower the user onboarding difficulty, and improve the user experience. For example, we could set up session keys for an on-chain game, which would allow us to get rid of the constant process of entering passwords, and we could also use session keys to make small, password-free payments.

In Polkadot.js there is a feature that allows you to choose to sign your password without having to enter it again within 15 minutes of entering it once, but as we can see by analyzing the source code, they do not use session keys but simply cache it for the UX boost. If we can use session keys wisely, we could have a much better user experience in our secondary wallet development!
