# Prepare an x402 exact transaction without signing

`x402::client::exact::prepare_unsigned_payment` accepts a buyer public
key and normalized `PaymentRequirements`, returning a `VersionedTransaction`.
It uses the same exact instruction builder as `build_payment`; no wallet,
`TransactionSigner`, payment header, submission or settlement is involved.
All required signature slots are zero-filled, including a separate sponsor slot.

```rust,no_run
use solana_pay_kit::x402::{
    client::exact::prepare_unsigned_payment,
    protocol::schemes::exact::PaymentRequirements,
};
use solana_pubkey::Pubkey;
use solana_rpc_client::rpc_client::RpcClient;

async fn prepare(
    buyer: &Pubkey,
    rpc: &RpcClient,
    requirements: &PaymentRequirements,
) -> Result<solana_transaction::versioned::VersionedTransaction,
            solana_pay_kit::x402::error::Error> {
    // Supply requirements.recent_blockhash to avoid an RPC lookup.
    let unsigned = prepare_unsigned_payment(buyer, rpc, requirements, None).await?;
    // Validate economic terms and the compiled message before handing it to a
    // separately controlled signer. Do not rebuild after approval/signing.
    Ok(unsigned)
}
```

Preparation preserves the exact builder's token/SOL instructions, amount,
recipient, mint, compute budget, memo, blockhash and fee-payer rules, as well as
transaction-version negotiation. It does not provide a new supported protocol.
If no seller memo is supplied, preparation generates a fresh memo just as the
existing builder does. Retain that transaction; two calls need not be identical.
If no blockhash is supplied, the existing RPC lookup still happens.

The result is not a payment proof. A caller may encode it with `core::tx::encode`
for transport, but it cannot authorize payment until the required parties sign.
This API does not enforce application policy, provide durable idempotency, revoke
signatures, or control final HTTP egress. A remote-signer callback alone cannot
provide those guarantees. Keep credentials within the chosen trust boundary.

Existing `build_payment` and `build_payment_header` signatures are unchanged.
`build_payment` prepares the same transaction, signs its buyer slot through the
existing signer, and returns the same encoded `PaymentPayload` as before.
