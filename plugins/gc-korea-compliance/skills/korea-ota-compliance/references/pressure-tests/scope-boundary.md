# Pressure Test: Correct Scope Boundary with Payment Platform

## Scenario

PR adding TossPay as a new payment method.

**New file — `PaymentMethodService.java`:**

```java
@Service
public class PaymentMethodService {

    public PaymentResult initiatePayment(PaymentRequest request) {
        PaymentGateway gateway = gatewayResolver.resolve(request.getPaymentMethod());

        PaymentGatewayRequest gatewayRequest = PaymentGatewayRequest.builder()
            .orderId(request.getOrderId())
            .amount(request.getAmount())
            .productName(request.getProductName())
            .customerName(request.getCustomerName())       // traveler name
            .customerEmail(request.getCustomerEmail())     // traveler email
            .paymentMethod(request.getPaymentMethod())
            .returnUrl(request.getReturnUrl())
            .build();

        return gateway.initiate(gatewayRequest);
    }
}
```

**New file — `TossPayGateway.java`:**

```java
@Component
public class TossPayGateway implements PaymentGateway {

    @Override
    public PaymentResult initiate(PaymentGatewayRequest request) {
        return tossPayClient.requestPayment(request);
    }
}
```

---

## Expected behavior

The skill must correctly apply scope boundaries:

1. **PCI-DSS is the payment platform's responsibility, not the OTA's.** TossPay handles card data. The OTA passes order metadata. Flagging PCI-DSS against this OTA code is a scope error.

2. **The data transfer to TossPay is legitimate but requires documentation.** `customerName` and `customerEmail` are personal data transferred to a third-party payment processor. This needs:
   - Classification: entrustment (수탁) vs. third-party provision (제3자 제공) — TossPay as payment processor is typically entrustment.
   - Transfer register entry.
   - Consumer disclosure at checkout (who receives their data and for what purpose).

3. **TossPay is a Korean company.** Do not treat this as an overseas transfer requiring cross-border PIPA notice.

### Expected finding

```
[WARN] PIPA-005 — Customer personal data (name, email) transferred to TossPay
for payment processing. Verify:
  - Transfer register updated with TossPay as entrustee (수탁자).
  - Checkout disclosure includes TossPay data handling.
  - Entrustment agreement (위탁 계약) in place with TossPay.
  - If classified as third-party provision, separate consent is required.
```

---

## Failure modes

- Flagging PCI-DSS compliance against OTA application code — **wrong scope**: PCI-DSS obligations sit with the payment platform, not the OTA integrating via API.
- Flagging the entire integration as BLOCK — **wrong severity**: data transfer to a payment processor is expected and legitimate; it needs documentation, not blocking.
- Not recognizing the data transfer at all and approving without any finding.
- Treating TossPay as an overseas recipient and applying cross-border transfer requirements.
- Demanding raw card-data handling controls from OTA code that never touches card data.

---

## Why this is hard

Payment integrations feel high-risk. A model primed on security may over-apply payment security rules — PCI-DSS, card data handling, tokenization requirements — to OTA application code that only passes order metadata to a payment gateway.

The OTA's actual obligations here are narrower:
- PIPA data flow documentation (transfer register, entrustment classification).
- Consumer disclosure at checkout.
- Not collecting or logging card data itself.

PCI compliance, card data security, and tokenization are the payment platform's domain. Recognizing this boundary, and issuing a calibrated WARN rather than a BLOCK, is the correct outcome.

This test verifies that the skill distinguishes between "this feels like a payment security issue" and "what does the OTA actually own here."
