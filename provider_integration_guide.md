# Provider Integration Guide

This guide explains how to integrate a new business payout provider into the Crypto Studio backend. Follow the same pattern as `TestProvider` (sandbox/reference) and `Sokin` (production example).

---

## Overview

A provider integration requires **4 files** inside `src/providers/business/<ProviderName>/`:

| File | Base Class | Purpose |
|------|-----------|---------|
| `<providername>_kyc.ts` | `IKyc` | Onboard the user/company with the provider (multi-step KYC) |
| `<providername>_wallet.ts` | `BaseWalletInterface` | Create/fetch a user wallet on the provider side |
| `<providername>_beneficiary.ts` | `BaseBeneficiaryInterface` | Register a payout recipient with the provider |
| `<providername>_payout.ts` | `BasePaymentInterface` | Execute payouts, fetch quotes, handle webhooks and status |

> **Naming rule:** The folder name and file prefix must be **lowercase** and must match exactly what is stored in the `providers.filename` database column.  
> `LoadProvider.business("sokin", "Payout")` loads `src/providers/business/sokin/sokin_payout.ts`.

---

## Folder Structure

```
src/providers/business/<ProviderName>/
  client.ts                      # (optional) API client / authentication helper
  utils.ts                       # (optional) mapping helpers, constants
  <providername>_kyc.ts
  <providername>_wallet.ts
  <providername>_beneficiary.ts
  <providername>_payout.ts
```

---

## Step 1 — KYC (`<providername>_kyc.ts`)

Extend `IKyc` from `src/modules/Kyc/BaseKycAbstruct.ts`.

```ts
import { IKyc } from "../../../modules/Kyc/BaseKycAbstruct.js";
import type {
  InitKycPayload,
  InitKycResponse,
  StatusKycPayload,
  StatusKycResponse,
} from "../../../types/kyc.types.js";
import { ConnectorOnboardingStatus } from "../../../generated/prisma/client.js";
import prisma from "../../../config/database.js";

export default class MyProvider_kyc extends IKyc {

  /**
   * Called for each KYC step submitted by the user.
   * `payload.step` (number) identifies which step is being submitted.
   * `payload.data` contains the user-submitted form data for that step.
   * `payload.connector` is the connector DB record (has `.id`, `.provider_id`, `.config`).
   */
  async init(payload: InitKycPayload): Promise<InitKycResponse> {
    const { step, user_id, data, connector } = payload;

    switch (step) {
      case 1:
        // Call provider API to create/register the corporate customer.
        // Store the provider's customer reference in userConnector:
        await prisma.userConnector.update({
          where: {
            user_id_connector_id: {
              user_id: Number(user_id),
              connector_id: connector.id,
            },
          },
          data: {
            provider_user_reference: "<provider_customer_id>",
            onboarding_status: ConnectorOnboardingStatus.INP_ROGRESS,
          },
        });
        return {};

      case 2:
        // Upload documents, etc.
        return {};

      default:
        // Final submission step — mark as SUBMITTED
        await prisma.userConnector.update({
          where: {
            user_id_connector_id: {
              user_id: Number(user_id),
              connector_id: connector.id,
            },
          },
          data: { onboarding_status: ConnectorOnboardingStatus.SUBMITTED },
        });
        return {};
    }
  }

  /**
   * Called to poll the KYC approval status from the provider.
   * Must return `{ status: "SUCCESS" | "PENDING" | "FAILED", reason?: string }`.
   */
  async status(payload: StatusKycPayload): Promise<StatusKycResponse> {
    // Call provider API to get approval status
    return { status: "SUCCESS", reason: "Approved" };
  }

  commonRoutes(router: any): void {
    // Register any provider-specific webhook routes here if needed
    // e.g. router.post("/myprovider/kyc-webhook", this.webhook);
  }
}
```

### Key points
- `provider_user_reference` stored in `userConnector` is the handle used in all future API calls on behalf of the user (see Sokin's `onBehalfOf` pattern).
- `onboarding_status` must progress: `INP_ROGRESS` → `SUBMITTED` → (webhook updates to `APPROVED` / `REJECTED`).

---

## Step 2 — Wallet (`<providername>_wallet.ts`)

Extend `BaseWalletInterface` from `src/modules/Payment/BasePaymentAbstract.ts`.

```ts
import {
  BaseWalletInterface,
  type WalletCreationPayload,
  type WalletCreationResponse,
} from "../../../modules/Payment/BasePaymentAbstract.js";

export default class MyProvider_wallet extends BaseWalletInterface {

  /**
   * Called when a user requests a new wallet for a given currency.
   * `payload.walletPayload` is the Wallets DB row being created.
   * `payload.connectorData` is the connector (contains `.config` with API credentials).
   *
   * Must return:
   *   provider_wallet_refrence  — the provider's ID for this account (stored in wallets table)
   *   banking_details           — account/IBAN/routing info shown to the user for funding
   *   extra_response_feild      — any additional provider data to persist
   */
  async createWallet(payload: WalletCreationPayload): Promise<WalletCreationResponse> {
    // 1. Authenticate with the provider using payload.connectorData.config
    // 2. Call provider API to create or fetch the currency account
    // 3. Return the references

    return {
      provider_wallet_refrence: "<provider_account_id>",
      banking_details: {
        iban: "...",
        account_number: "...",
        sort_code: "...",
        bank_name: "...",
      },
      extra_response_feild: {},
    };
  }

  async fetchWallet<T>(payload?: any): Promise<T> {
    // Optionally list wallets from the provider
    return [] as unknown as T;
  }
}
```

---

## Step 3 — Beneficiary (`<providername>_beneficiary.ts`)

Extend `BaseBeneficiaryInterface`.

```ts
import {
  BaseBeneficiaryInterface,
  type BeneficiaryCreationPayload,
  type BeneficiaryCreationResponse,
} from "../../../modules/Payment/BasePaymentAbstract.js";
import { BadRequestException } from "../../../utils/excepion.helper.js";

export default class MyProvider_beneficiary extends BaseBeneficiaryInterface {

  /**
   * Registers a beneficiary on the provider side.
   * `payload.beneficairy_data` is the Beneficiary DB row.
   * `payload.connectorData` has the connector config.
   * `payload.userId` is the user's internal ID.
   *
   * Must return:
   *   provider_banaficary_refrence — provider's beneficiary ID (cached in beneficiary_provider_ref table)
   */
  async createBeneficiary(payload: BeneficiaryCreationPayload): Promise<BeneficiaryCreationResponse> {
    // Map internal beneficiary fields to the provider's expected payload
    // Call provider API
    return {
      provider_banaficary_refrence: "<provider_beneficiary_id>",
      extra_response_feild: {},
    };
  }

  async updateBeneficiary(payload: any) {
    throw new BadRequestException("Update not supported. Contact support.");
  }

  async getBeneficiaryForm(country: string, type: string) {
    // Return form field definitions for the given country/type if needed
    return [];
  }
}
```

---

## Step 4 — Payout (`<providername>_payout.ts`)

Extend `BasePaymentInterface`. This is the most complex file.

```ts
import {
  BasePaymentInterface,
  type PaymentCheckoutPayload,
  type QuotePayload,
  type QuoteResponse,
  type StatusPayload,
} from "../../../modules/Payment/BasePaymentAbstract.js";
import type { ProviderResponseType } from "../../../types/payment.types.js";
import type { Request, Response, NextFunction, Router } from "express";
import { storeAndUpdateProviderTxnLog } from "../../../utils/payment.helper.js";
import { PaymentStatus, TransactionLog } from "../../../enums/payment.enums.js";
import { BadRequestException } from "../../../utils/excepion.helper.js";
import prisma from "../../../config/database.js";

export default class MyProvider_payout extends BasePaymentInterface {

  /**
   * Executes the actual payout via the provider API.
   * Called by `payoutProcessingWorker` (BullMQ) after the transaction is stored.
   *
   * Available inputs:
   *   payload          — full Transactions DB row (transaction_id, amount, currency, etc.)
   *   beneficiary_data — full Beneficiary DB row
   *   connector_data   — Connectors row including .config (API credentials)
   *   wallet_data      — Wallets row including .provider_wallet_refrence
   *   quoteData        — QuoteResponse from Redis (null if same-currency payout)
   *
   * Must return ProviderResponseType:
   *   { status: PaymentStatus, message: string, gateway_id: string }
   */
  async checkout({
    payload,
    beneficiary_data,
    connector_data,
    wallet_data,
    quoteData,
  }: PaymentCheckoutPayload): Promise<ProviderResponseType> {
    try {
      // 1. Get the user's provider reference
      const userConnector = await prisma.userConnector.findUnique({
        where: {
          user_id_connector_id: {
            user_id: payload.user_id,
            connector_id: connector_data.id,
          },
        },
      });

      // 2. Build payout payload for provider
      const payoutPayload = {
        // map fields from payload, beneficiary_data, wallet_data, quoteData
      };

      // 3. Call provider payout API
      // const response = await myProviderClient.payout(payoutPayload);

      // 4. Log the attempt
      await storeAndUpdateProviderTxnLog({
        transaction_id: payload.transaction_id,
        status: PaymentStatus.PENDING,
        message: "Payout initiated",
        gateway_id: "<provider_transaction_id>",
        mode: "Live",
        type: TransactionLog.CHECKOUT,
        response: {}, // raw provider response
        payload: payoutPayload,
      });

      return {
        status: PaymentStatus.PENDING,
        message: "Payout submitted successfully",
        gateway_id: "<provider_transaction_id>",
      };
    } catch (error: any) {
      throw new BadRequestException(error?.message || "Payout failed");
    }
  }

  /**
   * Get an FX quote before initiating a cross-currency payout.
   * Only needed when payload.currency !== payload.converted_currency.
   *
   * Must return QuoteResponse including `provider_quote_refrence`
   * which is passed back to `checkout` via `quoteData`.
   */
  async quote(data: QuotePayload): Promise<QuoteResponse> {
    try {
      // Call provider quote/FX API
      return {
        provider_quote_refrence: "<provider_quote_id>",
        from_currency: data.payload.from_currency,
        to_currency: data.payload.to_currency,
        source_amount: 0,
        destination_amount: 0,
        rate: 0,
      };
    } catch (e: any) {
      throw new BadRequestException(e?.message || "Quote failed");
    }
  }

  /**
   * Fetch the live status of a payout from the provider.
   * `gateway_id` is the provider's transaction reference.
   * `metaData` carries any extra context (e.g. userConnector record).
   */
  async status({ gateway_id, connector_id, metaData }: StatusPayload): Promise<any> {
    try {
      // Call provider status API
      return {};
    } catch (error: any) {
      throw new BadRequestException(error?.message || "Status fetch failed");
    }
  }

  /**
   * Handles incoming webhook events from the provider.
   * Must update the transaction status via storeAndUpdateProviderTxnLog.
   * Must respond with HTTP 200 to acknowledge receipt.
   */
  async webhook(req: Request, res: Response, next: NextFunction): Promise<void> {
    try {
      const { transaction_id, status } = req.body || {};
      // 1. Look up the transaction
      const transaction = await prisma.transactions.findFirst({
        where: { transaction_id: String(transaction_id) },
      });
      if (!transaction) throw new BadRequestException("Transaction not found");

      // 2. Map provider status to internal PaymentStatus
      const mappedStatus = PaymentStatus.SUCCESS; // apply your mapping logic

      // 3. Persist the status update
      await storeAndUpdateProviderTxnLog({
        transaction_id: transaction.transaction_id,
        status: mappedStatus,
        message: "Webhook received",
        mode: "Live",
        type: TransactionLog.WEBHOOK,
        response: req.body,
        payload: req.body,
      });

      res.status(200).json({ success: true });
    } catch (error) {
      next(error);
    }
  }

  /**
   * Register provider-specific public routes on the Express router.
   * These are mounted under /api/provider (or similar) and are unauthenticated.
   * Used for webhooks, redirects, simulation endpoints, etc.
   */
  commonRoutes(router: Router): void {
    router.post("/myprovider/webhook", this.webhook);
  }

  async redirect(req: Request, res: Response, next: NextFunction): Promise<void> {
    // Implement if provider uses redirect-based flow
  }
}
```

---

## Database Setup

### 1. Register the provider
Insert a row in the `providers` table:

```sql
INSERT INTO providers (name, filename, type)
VALUES ('My Provider', 'MyProvider', 'BUSINESS');
```

`filename` must match the folder name in `src/providers/business/`. This value is retrieved via `wallet.connector.provider.filename` and passed directly to `LoadProvider.business(filename, "Payout")`.

### 2. Create a connector
Insert a row in the `connectors` table:

```sql
INSERT INTO connectors (provider_id, name, config)
VALUES (
  <provider_id>,
  'My Provider - Live',
  '{
    "apiUrl": "https://api.myprovider.com",
    "client_id": "...",
    "client_secret": "...",
    "grant_type": "client_credentials",
    "audience": "..."
  }'
);
```

The `config` JSON is passed as `connector_data.config` to all provider methods.

### 3. Assign connector to currency
Link the connector to the appropriate business currency so that wallets are matched correctly in `PayoutController.getQuote` and `createPayout`.

---

## Payout Flow (End-to-End)

```
Client                     API                        BullMQ Worker            Provider API
  |                          |                              |                       |
  |-- GET /payout/quote ----->|                              |                       |
  |                          |-- LoadProvider.businessPayout|                       |
  |                          |-- providerInstance.quote() --|-- FX Rate API ------->|
  |                          |                              |                       |
  |<-- { quote_id, ... } ----|                              |                       |
  |                          |                              |                       |
  |-- POST /payout/create --->|                              |                       |
  |                          |-- Validate wallet, beneficiary                       |
  |                          |-- Validate quote from Redis  |                       |
  |                          |-- Check wallet balance       |                       |
  |                          |-- storeTransaction() ------->|                       |
  |                          |-- scheduleUserPayoutProcessingJob()                  |
  |<-- { transaction_id } ---|                              |                       |
  |                          |                              |                       |
  |                          |              (async) payoutProcessingWorker          |
  |                          |                              |-- LoadProvider        |
  |                          |                              |-- checkout()          |
  |                          |                              |-- Provider API ------->|
  |                          |                              |                       |
  |                          |              (webhook) POST /provider/webhook        |
  |                          |<-- webhook event ------------|-- provider sends ---->|
  |                          |-- webhook() handler          |                       |
  |                          |-- storeAndUpdateProviderTxnLog                      |
```

### Quote caching
- `getQuote` stores the full `QuoteResponse` (including `provider_quote_refrence`) in Redis with a 2-minute TTL.
- `createPayout` retrieves it by `quote_id` and validates that `from_currency`, `to_currency`, and `provider_quote_refrence` are present.
- The `provider_quote_refrence` is forwarded in `quoteData` to `checkout()` so the provider can link the payout to the locked rate.

---

## `LoadProvider` Resolution

`LoadProvider.business(providerFileName, moduleName)` dynamically imports:

```
src/providers/business/{providerFileName}/{providerFileName_lowercase}_{moduleName_lowercase}.js
```

Examples:
| Call | Resolves to |
|------|------------|
| `LoadProvider.businessPayout("sokin")` | `src/providers/business/sokin/sokin_payout.ts` |
| `LoadProvider.businessKyc("TestProvider")` | `src/providers/business/TestProvider/testprovider_kyc.ts` |
| `LoadProvider.businessWallet("MyProvider")` | `src/providers/business/MyProvider/myprovider_wallet.ts` |

> The class is instantiated with `new module()` — ensure each file has a **default export** that is a **class**.

---

## TestProvider vs Sokin — Design Differences

| Concern | TestProvider | Sokin |
|---------|-------------|-------|
| Auth | None | OAuth2 `client_credentials` via `sokinAuth()`, cached in Redis for 23h |
| Beneficiary creation | Returns a dummy reference | Calls `/beneficiaries` API, maps payload via `mapToSokinPayload()` |
| Wallet creation | Generates fake IBAN | Matches existing Sokin currency account by currency code |
| KYC steps | Single step, stores SUBMITTED | Multi-step: create customer → upload docs → upload associate docs → final submission |
| Quote | Calls public `open.er-api.com` | Calls Sokin `/fx/rate` on behalf of user |
| Payout | Simulates success, stores PENDING | Real `/instruction-requests/payment` or `/fx-payment` |
| Webhook | Accepts `{ transaction_id, status }` simulation payload | Handles Sokin `notificationType` events, fetches live status from provider |
| Status | Looks up DB only | Calls Sokin `/instruction-requests/{id}` or `/instructions/{id}` |

---

## Checklist for a New Provider

- [ ] Create folder `src/providers/business/<ProviderName>/`
- [ ] Add `client.ts` — authenticated API client factory
- [ ] Add `utils.ts` — status/field mappers
- [ ] Implement `<providername>_kyc.ts` extending `IKyc`
- [ ] Implement `<providername>_wallet.ts` extending `BaseWalletInterface`
- [ ] Implement `<providername>_beneficiary.ts` extending `BaseBeneficiaryInterface`
- [ ] Implement `<providername>_payout.ts` extending `BasePaymentInterface`
- [ ] Insert `providers` row in DB (`filename` = folder name)
- [ ] Insert `connectors` row with credentials in `config` JSON
- [ ] Link connector to business currency
- [ ] Register webhook route in `commonRoutes()` or via Express router
- [ ] Test with TestProvider pattern first (dummy responses), then swap to live API calls
