# Evidence

- `orders.csv`: every order on the Spout devnet orders program `SPoRXgsB4gWZmWPwyndoRWQrmZXKUc7o7oPMdkkGRcG` from 2026-06-08 to 2026-09-10 12:07 UTC (1,184 rows), matched to its fill where one exists. Built from public RPC reads (`getSignaturesForAddress`, `getTransaction`) on api.devnet.solana.com. Columns: place time and signature, side, ticker, mint, user, pending-order account, USDC and asset amounts, oracle price and staleness at placement, fill time and signature, latency, order id, minted or paid amounts versus quoted.
- `order-confirmed-market-closed.jpg`: the confirmation modal shown for the queued buy on 2026-09-10 12:35 UTC ("You own 0.04 NVDA", "Bought at $223.77", "Market is closed. Your order fills at 9:30 AM ET").

All addresses and signatures are public devnet state. The test wallet is a throwaway.
