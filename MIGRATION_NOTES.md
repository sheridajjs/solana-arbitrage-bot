# Jupiter V4 to V6 API Migration Notes

## Summary
This document describes the migration from Jupiter V4 SDK (`@jup-ag/core`) to Jupiter V6 API (`@jup-ag/api`).

## Key Changes

### 1. Package Dependencies (package.json)
- **Removed**: `"@jup-ag/core": "^4.0.0-beta.21"`
- **Added**: `"@jup-ag/api": "^6.0.0"`

### 2. Token List URL (src/wizard/Pages/Tokens.js)
- **Before**: Imported `TOKEN_LIST_URL` from `@jup-ag/core`
- **After**: Defined local constants for token list URLs:
  ```javascript
  const TOKEN_LIST_URL = {
    'mainnet-beta': 'https://token.jup.ag/strict',
    'devnet': 'https://token.jup.ag/devnet-strict'
  };
  ```

### 3. Jupiter Initialization (src/bot/setup.js)
- **Before**: Used `Jupiter.load()` with extensive configuration options
  ```javascript
  const jupiter = await Jupiter.load({
    connection,
    cluster: cache.config.network,
    user: wallet,
    restrictIntermediateTokens: false,
    shouldLoadSerumOpenOrders: false,
    wrapUnwrapSOL: cache.wrapUnwrapSOL,
    ammsToExclude: { ... }
  });
  ```
- **After**: Use `createJupiterApiClient()` and store connection/wallet in cache
  ```javascript
  const jupiterQuoteApi = createJupiterApiClient();
  cache.connection = connection;
  cache.wallet = wallet;
  ```
- **Note**: V6 API is a REST API client, so AMM exclusions are not configured at initialization time.

### 4. Route Computation (src/bot/index.js & src/bot/setup.js)
- **Before**: Used `jupiter.computeRoutes()` which returned `routesInfos` array
  ```javascript
  const routes = await jupiter.computeRoutes({
    inputMint: new PublicKey(inputToken.address),
    outputMint: new PublicKey(outputToken.address),
    amount: amountInJSBI,
    slippageBps: slippage,
    forceFetch: true,
    onlyDirectRoutes: false,
    filterTopNResult: 2,
  });
  const route = routes.routesInfos[0];
  ```
- **After**: Use `jupiter.quoteGet()` which returns a quote response directly
  ```javascript
  const quoteResponse = await jupiter.quoteGet({
    inputMint: inputToken.address,
    outputMint: outputToken.address,
    amount: amountToTrade.toString(),
    slippageBps: slippage,
    onlyDirectRoutes: false,
    maxAccounts: 64,
  });
  const route = quoteResponse;
  ```
- **Key Differences**:
  - Input/output mints are strings (addresses), not PublicKey objects
  - Amount is a string, not JSBI.BigInt
  - Returns single best quote, not array of routes
  - Response has `routePlan` instead of multiple `routesInfos`

### 5. Swap Execution (src/bot/swap.js)
- **Before**: Used `jupiter.exchange()` which returned an `execute` function
  ```javascript
  const { execute } = await jupiter.exchange({
    routeInfo: route,
    computeUnitPriceMicroLamports: priority,
  });
  const result = await execute();
  ```
- **After**: Use `jupiter.swapPost()` to get transaction, then sign and send manually
  ```javascript
  const swapResult = await jupiter.swapPost({
    swapRequest: {
      quoteResponse,
      userPublicKey: cache.wallet.publicKey.toBase58(),
      wrapAndUnwrapSol: cache.wrapUnwrapSOL,
      computeUnitPriceMicroLamports: priority,
      prioritizationFeeLamports: 'auto',
    },
  });
  
  const swapTransactionBuf = Buffer.from(swapResult.swapTransaction, 'base64');
  const transaction = VersionedTransaction.deserialize(swapTransactionBuf);
  transaction.sign([cache.wallet]);
  
  const rawTransaction = transaction.serialize();
  const txid = await cache.connection.sendRawTransaction(rawTransaction, {
    skipPreflight: true,
    maxRetries: 2
  });
  
  await cache.connection.confirmTransaction({...}, 'confirmed');
  ```

### 6. Data Structure Changes
- **Route object fields**:
  - `route.amount` → `route.inAmount`
  - `route.outAmount` remains same but is now a string instead of JSBI
- **JSBI removal**: Native BigInt is used instead of JSBI for amount handling
  - `JSBI.BigInt(amount)` → `amount.toString()` for API calls
  - `JSBI.toNumber(route.outAmount)` → `BigInt(route.outAmount).toString()` for calculations

### 7. Response Format
- **Before**: Result included `txid`, `inputAmount`, `outputAmount`, `error`
- **After**: Same structure maintained for compatibility, constructed manually:
  ```javascript
  const result = {
    txid,
    inputAmount: quoteResponse.inAmount,
    outputAmount: quoteResponse.outAmount,
    error: null
  };
  ```

## Migration Benefits
1. **Better Performance**: REST API is more lightweight than SDK initialization
2. **Simpler Setup**: No need to pre-configure AMM exclusions
3. **Modern Standards**: Uses native BigInt instead of JSBI
4. **Better Control**: Manual transaction signing and sending provides more flexibility
5. **API Stability**: V6 is the current stable API version

## Testing Recommendations
1. Verify configuration wizard loads token list correctly
2. Test bot initialization with both strategies (pingpong and arbitrage)
3. Confirm route computation works for different token pairs
4. Validate swap execution completes successfully
5. Check error handling for failed swaps

## Important Notes
- The V6 API does not support AMM exclusion configuration at the client level. If AMM filtering is needed, it should be done at the route selection level.
- Transaction confirmation is now explicit and can be customized
- Priority fees are handled differently - now supports 'auto' mode
- All API calls now use REST endpoints instead of SDK methods
