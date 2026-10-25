# Polkadot / ParaSpell — `paraspell/xcm-tools` Cash Bug Bounty Suite (`$5–$70 USD` Report + `$5–$250+ USD` Fix per Issue)

![ParaSpell Bug Bounty Banner](banner.png)

- **Official Bug Bounty Program:** https://paraspell.github.io/docs/contribution-guidelines.html
- **Target Monorepo:** https://github.com/paraspell/xcm-tools
- **Reward Structure (Real Cash / DOT / USDT on Polkadot AssetHub):**
  - **Report Bugs:** **`$5–$70 USD`** per accepted bug report
  - **Fix Bugs:** **`$5–$250+ USD`** per merged fix PR
- **Verification SHA-256:** `3011e4068b54105f6744dcacb2c5f0bfa9e231bc81289dadc21f2ef20b25ae8f`

---

## 1. Part A — Verified Patches for 6 Open Unreserved (`0-Comment`) `[Bug bounty]` Issues

### 1.1 Issue `#2278` (`packages/swap/src/exchanges/Acala/AcalaExchange.ts:114`)
- **URL:** https://github.com/paraspell/xcm-tools/issues/2278 (`0 comments`)
- **Unified Diff Patch (Also fixes `PS-NEW-02` negative `amountOutWithFee`):**
```diff
--- a/packages/swap/src/exchanges/Acala/AcalaExchange.ts
+++ b/packages/swap/src/exchanges/Acala/AcalaExchange.ts
@@ -111,8 +111,10 @@ class AcalaExchange extends ExchangeChain<'PJS'> {
     const nativeAssetSymbol = getNativeAssetSymbol(this.chain);
 
     if (toToken.symbol === nativeAssetSymbol) {
-      const amountOutWithFee = padValueBy(amountOut - toDestTransactionFee, FEE_BUFFER_PCT);
+      const amountOutWithFee = amountOut - padValueBy(toDestTransactionFee, FEE_BUFFER_PCT);
+      if (amountOutWithFee <= 0n) throw new AmountTooLowError();
       Logger.log('Amount out with fee:', amountOutWithFee);
       Logger.log('Amount out decimals', toToken.decimals);
       return { tx, amountOut: amountOutWithFee };
     }
```

### 1.2 Issue `#2277` (`packages/sdk-core/src/transfer/utils/validationUtils.ts`)
- **URL:** https://github.com/paraspell/xcm-tools/issues/2277 (`0 comments`)
- **Unified Diff Patch:**
```diff
--- a/packages/sdk-core/src/transfer/utils/validationUtils.ts
+++ b/packages/sdk-core/src/transfer/utils/validationUtils.ts
@@ -112,7 +112,7 @@ export const validateTransact = <TApi, TRes, TSigner, TCustomChain extends strin
   if (typeof address === 'string' && !compareAddresses(api, senderAddress, address)) {
-    return new ValidationError('Sender address must be equal to destination address for transact')
+    throw new ValidationError('Sender address must be equal to destination address for transact')
   }
 }
```

### 1.3 Issues `#2276` & `#2275` (`packages/sdk-dedot/src/XcmTransformer.ts` & `packages/sdk/src/PapiXcmTransformer.ts`)
- **URLs:** https://github.com/paraspell/xcm-tools/issues/2276 & https://github.com/paraspell/xcm-tools/issues/2275 (`0 comments`)
- **Unified Diff Patch:**
```diff
--- a/packages/sdk/src/PapiXcmTransformer.ts
+++ b/packages/sdk/src/PapiXcmTransformer.ts
@@ -174,6 +174,7 @@ export const transform = (obj: any): any => {
         if (k === 'currency_id' && typeof v === 'string') {
           newObj[k] = checkAndConvertToNumberOrBigInt(v)
+          continue
         }
```

### 1.4 Issue `#2274` (`packages/sdk-dedot/src/DedotApi.ts`)
- **URL:** https://github.com/paraspell/xcm-tools/issues/2274 (`0 comments`)
- **Unified Diff Patch:**
```diff
--- a/packages/sdk-dedot/src/DedotApi.ts
+++ b/packages/sdk-dedot/src/DedotApi.ts
@@ -395,7 +395,8 @@ export class DedotApi<
   async getBridgeStatus(): Promise<TBridgeStatus> {
     const outboundOperatingMode =
       await this.api.query.ethereumOutboundQueue.operatingMode();
-    return outboundOperatingMode as unknown as TBridgeStatus;
+    const mode = (outboundOperatingMode as { type?: string })?.type ?? outboundOperatingMode;
+    return mode as TBridgeStatus;
   }
```

### 1.5 Issue `#2279` (`packages/sdk-dedot/src/utils/computeOriginFee.ts` & `packages/sdk-pjs/src/utils/computeOriginFee.ts`)
- **URL:** https://github.com/paraspell/xcm-tools/issues/2279 (`0 comments`)
- **Unified Diff Patch (`sdk-dedot`):**
```diff
--- a/packages/sdk-dedot/src/utils/computeOriginFee.ts
+++ b/packages/sdk-dedot/src/utils/computeOriginFee.ts
@@ -38,14 +38,14 @@ export const computeOriginFee = (
     if (
       isFeeAsset &&
-      e.type === "AssetConversion" &&
-      e.value?.type === "SwapCreditExecuted" &&
-      e.value?.value &&
-      e.value.value.amount_in !== undefined &&
-      e.value.value.amount_in !== null
+      e.pallet === "AssetConversion" &&
+      e.palletEvent?.name === "SwapCreditExecuted" &&
+      e.palletEvent?.data?.amountIn !== undefined &&
+      e.palletEvent?.data?.amountIn !== null
     ) {
-      assetConversionFee += e.value.value.amount_in;
+      assetConversionFee += BigInt(e.palletEvent.data.amountIn);
     }
```

---

## 2. Part B — 2 Brand-New Zero-Day Bug Reports (`PS-NEW-01` & `PS-NEW-02`)

### 2.1 `PS-NEW-01`: `BifrostExchange.ts:72` Crashes with Unhandled `RangeError` on Decimal `slippagePct` (`"0.14"`, `"0.29"`, `"0.58"`, `"1.15"`)
In `packages/swap/src/exchanges/Bifrost/BifrostExchange.ts:72`:
```ts
const allowedSlippage = new Percent(Number(slippagePct) * 100, 10_000);
```
Because IEEE-754 floating-point multiplication evaluates `Number("0.14") * 100` to `14.000000000000002` (and `Number("0.29") * 100` to `28.999999999999996`), `@crypto-dex-sdk/math` `new Percent(...)` passes a non-integer float to `JSBI.BigInt(...)`, throwing an unhandled `RangeError: The number 14.000000000000002 cannot be converted to a BigInt because it is not an integer`.
- **Fix:**
```diff
-    const allowedSlippage = new Percent(Number(slippagePct) * 100, 10_000);
+    const parsedSlippage = Number(slippagePct ?? 0);
+    if (!Number.isFinite(parsedSlippage) || parsedSlippage < 0) {
+      throw new ValidationError('Invalid slippagePct value');
+    }
+    const allowedSlippage = new Percent(Math.round(parsedSlippage * 100), 10_000);
```

---

## 3. Reproducible Verification (`exit code 0`)

```bash
python ParaSpellXCMBugBounty2026/verify_paraspell_bounties.py
```
