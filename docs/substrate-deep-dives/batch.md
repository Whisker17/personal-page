---
id: batch
title: Batch Pallet
tags:
  - Substrate
  - Deep Dives
---

Users are frustrated with the constant signing of on-chain operations, which can require multiple signatures in a short period of time for complex on-chain operations. We can choose to use Session keys to simplify this operation, but we also need to take the risk of doing so. Another simpler way is to do **Batch calls** for such operations, which require only one signature operation to complete all the steps.

## What are the batch calls in Substrate

Polkadot/Substrate provides a `utility.batch` method that can be used to send multiple transactions at once. These are then executed sequentially from a single sender (the specified single nonce). This is useful in many situations, for example, if you wish to create expenses for validators of multiple eras. Similarly, you can send multiple transfers at once. Even batch processing of different types of transactions.

But we have a corresponding restriction: **Batch dispatch is a stateless operation**. And it's allowed any origin to make multiple calls in a single dispatch.

## Code implementation

We limit the number of batch calls with `batched_calls_limit`

```rust
fn batched_calls_limit() -> u32 {
 let allocator_limit = sp_core::MAX_POSSIBLE_ALLOCATION;
 let call_size = ((sp_std::mem::size_of::<<T as Config>::RuntimeCall>() as u32 +
 CALL_ALIGN - 1) / CALL_ALIGN) *
 CALL_ALIGN;
 // The margin to take into account vec doubling capacity.
 let margin_factor = 3;
 allocator_limit / margin_factor / call_size
}
```

This is a Rust function called `batch` which takes two arguments, `origin` and `calls`. The `origin` argument is of type `OriginFor<T>`, which is the trait of the origin of the call, and `calls` is a vector of runtime calls.

```rust
pub fn batch(
origin: OriginFor<T>,
calls: Vec<<T as Config>::RuntimeCall>,
) -> DispatchResultWithPostInfo {
// Do not allow the `None` origin.
    if ensure_none(origin.clone()).is_ok() {
        return Err(BadOrigin.into())
    }

    let is_root = ensure_root(origin.clone()).is_ok();
    let calls_len = calls.len();

    ensure!(calls_len <= Self::batched_calls_limit() as usize, Error::<T>::TooManyCalls);

    // Track the actual weight of each of the batch calls.
    let mut weight = Weight::zero();

    for (index, call) in calls.into_iter().enumerate() {
        let info = call.get_dispatch_info();
        // If origin is root, don't apply any dispatch filters; root can call anything.
        let result = if is_root {
            call.dispatch_bypass_filter(origin.clone())
        } else {
            call.dispatch(origin.clone())
        };
    
        // Add the weight of this call.
        weight = weight.saturating_add(extract_actual_weight(&result, &info));
        if let Err(e) = result {
            Self::deposit_event(Event::BatchInterrupted {
                index: index as u32,
                error: e.error,
            });
        
            // Take the weight of this function itself into account.
            let base_weight = T::WeightInfo::batch(index.saturating_add(1) as u32);
            
            // Return the actual used weight + base_weight of this call.
            return Ok(Some(base_weight + weight).into())
        }
        Self::deposit_event(Event::ItemCompleted);
    }
    Self::deposit_event(Event::BatchCompleted);
    let base_weight = T::WeightInfo::batch(calls_len as u32);
    Ok(Some(base_weight + weight).into())
}
```

The purpose of this function is to execute a batch of runtime calls as a single atomic transaction. It checks if `origin` is `None` and then verifies that it is the root origin. If it is not the root origin, it applies a dispatch filter to that call. It then sums the weights of each call and checks for errors in the call. If there is an error, it returns the actual weight used plus the base weight of that call. If there are no errors, it returns the actual weight used plus the basic weight of the batch.

In addition to the most basic `batch` function, we also implement:

* `batch_all`, send a batch of dispatch calls and atomically execute them.

* `force_batch`, which allows errors to exist and does not interrupt.

## Summary

This is the breakdown of Batch calls in Substrate, very simple but I believe a very important part of a complete account system.
