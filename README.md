# I spent an evening on the Spout Finance testnet. Here is what I found.

**Update, 23 September 2026.** The [Borrow P0](#p0-borrowing-the-products-headline-does-not-work-from-the-app) below is out of date. By 20 September the app's deposit and borrow endpoints were answering with a borrow plan instead of the 500 error, and they still do today. On chain, three devnet wallets opened a vault, deposited collateral and borrowed in one transaction each on 18 and 19 September, and two repayments went through on the 18th. I have not rerun the borrow flow myself; the rest of the report is as tested on 10 September.

Spout Finance is building a Solana brokerage where you buy tokenized US stocks, borrow stablecoins against them at 0% interest, and lend stablecoins into the pool that funds those loans. The zero rate is paid for by writing covered calls on the collateral. I went through their devnet beta on 10 September 2026 as part of the Superteam Earn beta intelligence challenge, with a throwaway wallet, faucet money, and the intent to break things. This is the write-up.

Short version. The beta is a real on-chain product with a real order pipeline behind it, and the team has done more work than the surface suggests. Five defects would stop a real user today. The wallet cannot be reconnected after a page reload or a closed tab. On Brave Wallet, with the wallet on its default network, every buy dies with an unreadable toast, because the app never checks or explains which network the wallet is on. Apple is quoted 28% below the price orders execute at, so a buyer gets fewer shares than the order screen promises. The Cancel button on an open order accepts the request and does nothing, which the chain shows has been true since June. And borrowing, the feature in the headline, is a form that cannot submit: the app's deposit and borrow endpoints answer all eleven assets with a decoding error. Orders that do go through fill at an oracle price the buyer never sees, a little worse than the quote nearly every time, and orders placed outside US hours wait for the open while the USDC has already moved to a treasury wallet. Lending is not live. The model itself is coherent, but the documents describing it disagree with each other and with the chain on who bears assignment risk, when liquidations run, and how compliance is enforced.

## How I tested

- Environment: beta.spout.finance, devnet build, version v1.0.0 (5d92f4c) per the About page. Two sessions on 10 September; the US market was closed for all but the last half hour, which matters for order handling.
- Wallet: a throwaway Brave Wallet account, connected through the app's Privy login with a Sign-In-With-Solana message. Funded with 7 devnet SOL and 20 devnet USDC from Circle's faucet. The app's USDC is Circle's devnet mint, so the faucet route works. The wallet sat on Solana mainnet, the default, until I found out why that mattered.
- Browser: Brave on Linux, desktop width. I read the app's own network calls, the wallet's error objects, and on-chain accounts rather than guessing. The complete history of the orders program since June, 2,369 transactions and 1,184 orders from 42 wallets, is the basis for every claim about fills and cancels.
- What I exercised: the gate, the tour, wallet connect and reconnect, a buy placed with the market closed and filled at the open, a cancel, a sell filled with the market open, and the Borrow page up to its dead button. What I could not: a borrow, a repay, a health-factor change, a liquidation, or lending, because the Borrow API is down and Earn is not live. The leverage slider on the buy panel never responded either. That is the limit of this report.

## Bugs, ranked

### P0. A page reload strands the user

Log in, connect Brave Wallet, sign the message, trade page works. Reload the tab. The header goes back to "Connect", and the button does nothing. No modal, no error, no console output, no network call. The bottom "Connect Wallet" button is equally dead. I reproduced this five times: minutes after a fresh login, with Brave Shields on and off, by script and by hand, after closing the tab in the middle of a login, and after typing a URL into the address bar.

What is happening, from the browser's storage: Privy's session tokens survive the reload, so the app believes the user is authenticated, but the external wallet connection list is empty and is never re-established. My reading is that the Connect button calls the login flow, which is a no-op for an authenticated user. The one thing that would help, a call to connect a wallet to the existing session, never happens.

The explicit Disconnect in Settings does the right thing: it clears the tokens, the header flips to Connect, and Connect opens the Privy modal again. So the fix is narrow: on session restore, either re-attach the wallet or route Connect to the wallet-connect flow instead of login. Until then the only way back in is clearing site data.

### P0. On Brave Wallet, every buy dies with the text "[object Object]"

Set an amount, click "Buy 0.04 NVDA". The button dims for three seconds, a toast flashes the literal text "[object Object]", and the button comes back. Nothing else changes: the USDC balance stays, the orders list is empty, the portfolio shows no holdings, and no signing prompt ever appears. Six attempts across two sessions, identical.

The server side is fine. The POST to the buy endpoint returns 200 with an order id, a fully built unsigned transaction, a blockhash, and a preflight result with no blockers and no warnings. The transaction escrows the user's USDC into Spout's USDC account and opens a pending order; the shares arrive after the fill. That is a sensible design for a broker-settled asset.

The client then hides what happens next. I logged every object the page turned into a string during the click, and the only one was the wallet's answer: error code -32603, "Blockhash is invalid or can not be validated". That is what Brave Wallet returns when a transaction's blockhash does not exist on the network the wallet is set to. My wallet was on Solana mainnet, which is the default. The app builds devnet transactions. Switching the wallet to devnet by hand made the same click open a signing prompt.

The bug is in the app around the transaction. It never checks which network the wallet is on, never tells a Brave user to switch, and renders the wallet's error object as a string instead of showing its message. On mainnet the default network will match and this particular failure disappears; what stays is an app that swallows wallet errors and records nothing when an order fails. The chain shows other testers placing orders all day, 117 orders from 21 wallets in three days, so most people find a way through, and the ones who do not have nothing to report. Fixes: check the wallet's network before building the order and say what to switch, show the error message text, and record a failed attempt in Activity so a user knows nothing was placed.

### P0. Apple is quoted 28% below the price your order executes at

The list and the detail page show AAPL at $228.50, up $0.96. The app's own price endpoint says $315.42 as of the 9 September close, and the chart tooltip on the same page shows $315.42 too. The cause is visible in the requests: the list page batches ten symbols in one price call and AAPL is not in the batch, so its card renders from a stale or seeded value.

On its own that is a display bug. It becomes a P0 because the order panel prices the order on the wrong number. With AAPL selected, $10 quotes "Buy 0.04 AAPL", and in share mode 0.05 shares quotes $11.43, which is $228.50 a share. The orders program does not use that price. It prices every order from the Stork feed at execution, and the Apple feed on devnet read $318.20 while I was testing. I did not have to guess what happens next, because other testers have already bought Apple: seven AAPL orders between 7 and 9 September were priced at $313.45 to $320.05 when placed and filled between $314.70 and $318.30, and a $10 order on 9 September at 18:50 UTC minted 0.0315 shares. The panel would have shown that buyer 0.04. They paid what the screen said and received 28% fewer shares than it promised. To be exact about the harm: the buyer still gets $10 of Apple at a fair oracle price, so nobody loses money on the fill. What they lose is the decision, made on a number 28% wrong, with no step in between that shows the real price.

Nothing in the flow catches this: no execution price on the confirm button, no slippage bound, no warning when a quote is older than the last close. Fix the batch request today. Then show the oracle price and its age on the button before a user signs, and reject an order when the panel's price and the oracle diverge beyond a set tolerance.

### P0. Cancel accepts your request and does nothing, and never has

Portfolio lists an open order with a Cancel button. I pressed it on my own order. The request returned 200, the row changed to "Cancelling…", no wallet prompt, no dialog. Forty seconds later Spout's sweep moved my 10 USDC from the escrow to its treasury, as it does for every order. After I logged in again the same row showed no status at all and the Cancel button was live again, and the app's own status feed for the order listed "received" and "accepted" with no cancel event. At the open the keeper filled the order I had cancelled, 58 seconds after the bell, and the row and its "Cancelling…" state vanished as if nothing had been asked.

The chain explains why. The orders program does have a RequestCancelOrder instruction, and ten transactions have called it since June, from four wallets. Every one writes the log line "Cancel requested by user" and moves nothing: no refund, no account closed, no lamports beyond the fee. Every order those ten calls targeted is still open today, weeks later, and two of the four wallets tried again, two and three times, which is what a person does when a button does nothing. My own cancel did not even reach the chain. In its whole history the program has never closed an order any way except a keeper fill, and 43 orders placed before the 24 August relaunch, 52 USDC of buys and fifteen sells' worth of tokens, have sat open for three to thirteen weeks with no path out.

On devnet this is faucet money. A cancel that is accepted and then ignored is worse than no cancel, because the user stops watching. Either wire RequestCancelOrder to a refund from the escrow before the sweep, or remove the button and say that orders are final once signed.

### P0. Borrowing, the product's headline, does not work from the app

With the NVDA shares in the wallet, the Borrow page finally has something to show: NVDA at $9.90, max borrow $4.95, a panel with an LTV slider, a health-factor bar, "Annual Interest 0% (always)", and a button reading "Borrow $4.00 USDC". Under the button, in small red type, sits a raw error: "CollateralType: unexpected length 213 (expected 165, or 149 pre-migration)". Clicking the button does nothing at all. No toast, no dialog, no wallet prompt.

The error is the server's. The endpoints the page calls to build a deposit or a borrow transaction answer 500 with that exact message for all eleven assets; I asked each one. Read plainly: the vault's on-chain collateral account is now 213 bytes and the API only knows the two older layouts. The vault program's history says when that happened. On 9 September at 00:29 UTC the keeper ran about seventy admin transactions, initialising collateral pools and migrating vault positions, and the Borrow page has been a form that cannot submit since. The program itself still works: at 12:25 UTC on 10 September another wallet deposited GOOG and borrowed 2 USDC against it through some client other than this page. The app's route to the vault is what is broken, for every asset it lists.

Two smaller things in the same panel, for when it is fixed. It defaults to "Borrow against XOM" for a user who owns only NVDA. And the health factor it shows is collateral divided by debt, 2.00 at the maximum loan, while the docs define it as collateral times the asset's liquidation threshold divided by debt, which for NVDA at 58.8% is 1.18. A user reading 2.00 thinks a 50% drop is survivable. The docs say 8.8%.

### P1. What the order actually does, which the app never shows

Reading the orders program's history for 7 to 10 September, 252 transactions, taught me more about the order flow than the UI does.

- Orders execute at the live Stork oracle price at fill time, not at the quote on screen. The list shows the last close (NVDA $223.77, GS $1,029.18); the orders placed since that close carried oracle prices of $223.06 to $223.62 for NVDA and $1,029.42 to $1,033.00 for GS. The oracle staleness logged on orders ran from 3 seconds to just under five minutes. None of this is visible to the buyer. My own $10 order collected four prices: the panel said $223.77, the chain recorded it at $221.61, the fill worked out to $220.74, and the history row says $220.52. The confirmation should show the price the order will use and how old it is.
- Fast fills land a little short of the quote, nearly every time. For the 27 buys in the window filled within a minute of placing, the minted amount was below the placed quote every time, by 0.13% to 0.71%, median 0.36%. Since 25 August it is 189 of 194 fast buy fills short and 5 a hair above; fast sells have gone both ways. Prices do not move against the buyer 189 times out of 194 inside ten seconds, so this looks like a systematic fill-side rule, a fee or a spread or both. The docs promise a flat 0.20% fee, the place instruction charges nothing on chain, and the UI has no fee line. Whatever is taken, nothing discloses it.
- Orders placed outside US market hours queue for the open, and the user is told exactly once. The confirmation modal ends with "Market is closed. Your order fills at 9:30 AM ET", the one true line in the flow. The same modal opens with "Your purchase has been confirmed", "You own 0.04 NVDA" and "Bought at $223.77". None of that is true yet: nothing has filled, and the chain recorded my order at the oracle's $221.61, not the panel's $223.77. Afterwards Activity and Transaction History both show the order as "Executing" with the market closed, the history row says "Fees --", and after a re-login the Open Orders row shows no status at all. Fills are done by a Spout keeper key: 5 to 11 seconds after an order when the market is open, and for anything placed after the close, within three minutes of the next 9:30 ET open. That part works. All 58 off-hours orders in the window filled at the next open, including the ones placed on Labor Day, which waited for Tuesday. The escrowed USDC leaves the wallet immediately and is swept to a Spout treasury wallet about two and a half minutes later, whether or not the order has filled, by a plain token transfer signed from an off-chain key. A "queued until 9:30 ET" state with the price rule spelled out would fix this half. The cancel button is its own story, above.
- After the fill, the screens disagree about what I own. The Trade page says I own "0.05" NVDA, which is 0.0453 rounded up. Portfolio said "$0.00", "No Holdings" and "No Positions" for the several minutes after the fill, while the app's own balances endpoint already returned the 0.0453 NVDA; it caught up after my next order.
- Selling works, and is the fastest thing in the app. My sell of 0.0183 NVDA was placed at 13:40:16 UTC at the oracle's $218.81 and paid out nine seconds later, 3.998976 USDC against a 3.999999 quote; another tester's sell on 9 September paid 0.11% under its quote. Then the confirmation modal said "Your purchase has been confirmed", "You own 0.02 NVDA", "Bought at $219.09", "Total paid $4.00". It is the buy modal with the sell's numbers in it. The Trade page kept showing "0.05" shares owned after the sale left me with 0.027.

### P1. The product, the tour, the docs and the chatbot describe four different products

- Earn page: "Earn is coming soon. Lending vaults are on the way."
- The onboarding tour, steps 8 and 9: pick Senior or Junior, tap "Compare Senior vs Junior". Neither of those exists in the app.
- Docs: lending is live, Senior about 9% and Junior about 32%, distributions every Monday.
- Ask Spout, when asked "Is this real money?": "No. This is a demo. The wallet connection is simulated, the balances are sample data." The wallet connection is not simulated; I signed a real message and the app built a real devnet transaction. Asked whether a pending order can be cancelled, it explained how to borrow. Asked what happens to a buy placed while the market is closed, it recited the liquidation-hours paragraph.

One source of truth, versioned with the app, would remove most of this.

### P2. Everything else I hit

- Chart range buttons (1D to 1Y) do nothing. The page fetches one year of daily bars once and shows the same series under every label.
- The sort control says "Sort: Price" on narrow screens and "Newest" on wide ones. The list is alphabetical by ticker either way, with AAPL appended last.
- Tour step 3 anchors to the asset list, scrolls the page and covers the very card it is pointing at. Step 5 reads "0%interest" with the space missing.
- The leverage slider did not respond to a drag or to arrow keys. Either it is disabled for a wallet with no position, without any disabled state, or it is broken.
- The amount field silently rewrites "-5abc" to $5 and quotes a buy. 0.001 collapses the panel to "Enter Amount" with no minimum shown.
- The orders, positions and balances endpoints answered for my address with no session token at all; I did not try other addresses. Transactions history is correctly gated. If positions are user data, the rest should be gated too.
- Docs link to app.spout.finance, which does not load. The GitBook at spout.gitbook.io describes a 2025 bond-ETF product with a Q3 2025 roadmap. The fees page says "no origination fee"; the "who this is for" page mentions "the one-time origination fee on the borrow".
- The docs' server-rendered HTML drops every dollar figure that begins with $1 or $2. Fetched without a browser, the tranche example reads "In a 0m pool" and the liquidation example "NVDA falls again, to 02". A browser repairs it on hydration, so most readers never see it, but reader mode, crawlers and any text extraction (including whatever feeds Ask Spout) get broken numbers.

### What works well

Credit where it is due, because a bug list makes any beta look worse than it is.

- The insufficient-balance modal knows it is on devnet, says so in plain words, and links to the Circle and Solana faucets. It is the best piece of copy in the app.
- The wallet picker is broad: Brave Wallet, Phantom, Solflare, Backpack, Jupiter, WalletConnect, plus email login through Privy.
- Settings already scaffold the boring things that matter: CSV export "for tax filing", a year-end summary with dividends and withholding, a cost-basis statement, and notification toggles for cycle settlement, assignment and health-factor alerts.
- Ask Spout answers "what does this cost me" honestly: no interest, capped upside on the collateral, a 20% protocol cut of premium, typically under 1% a year for the borrower. The marketing does not say that; the chatbot does.
- Disconnect asks for confirmation and explains that positions stay on chain.
- Sells settle in nine seconds during market hours, at the oracle price; mine paid out within 0.03% of the quote.
- The keeper is reliable. The most recent 12,000 of its transactions have zero failures, oracle pushes run around the clock, in-hours fills take about eight seconds, and since the 24 August relaunch every off-hours order has filled at the next open, bar one filled early. The orders program has never had a failed transaction on devnet. The rent for each pending-order account is paid by Spout's vault, so a user pays only the network fee.

## Friction, in the order a new user meets it

The gate takes an email and a code, and it remembers you, which is good. Then the tour starts on its own, eleven steps, before you have connected anything. It is well written, but it demonstrates an order panel you cannot see on a narrow screen until you connect, and it describes a lending page that says "coming soon". Let the user connect first, then offer the tour.

Connecting is clean when it works: Privy modal, pick a wallet, sign a message. Nothing tells you that a KYC identity was just created for you. On mainnet that step will be real, so the beta is the place to explain what "identity account found" means and what happens if it is not found.

The trade list is readable. "Est. Borrow cost x%/yr" sits next to a headline that says 0% forever, with no tooltip, and the numbers are placeholders (more on them in the economics section).

The order panel rounds shares to two decimals, so $10 of NVDA reads as 0.04 shares while the real figure is 0.0447, and says nothing about the market being closed until after you sign.

The chatbot is keyword retrieval rather than comprehension. Asked "is lending live right now, and what is the liquidation threshold for NVDA", it returned a generic paragraph about the health factor and answered neither. Its canned answers are good; its free-text answers are not.

### The Borrow page, as far as it goes

The sponsor asked for a hard look at the borrowing UX and collateral management, so here is what the page offers once you hold a share. A collateral table lists all eleven assets with position value, shares owned, a max-borrow column at 50% LTV, a 30-day sparkline and a "See collateral detailed view" link per row. The panel on the right takes an amount or an LTV slider from 0% to 50%, shows "You hold 0.045302013 NVDA / $10", a health-factor bar, "Est. borrower cost/yr", "Annual Interest 0% (always)" and "You receive". Below it sit "Active Positions" and the empty-state line "Deposit collateral to open a position". Settings already has toggles for health-factor and assignment alerts. That is a reasonable skeleton.

What is missing is everything a borrower needs before signing: the liquidation price for this loan, the asset's buffer and fee, whether a closed market changes anything, the assignment mechanics in one sentence, and any visible path to add collateral, repay, or unlock. "Deposit collateral to open a position" implies a separate lock step that the docs describe and the page never shows. Because the API is down I could not reach the position view, so I cannot say whether repay and unlock exist further in. Until the API works, this page is a preview.

## What I would ship before public launch, in order

1. Session restore. Re-attach the wallet on load, or make Connect call the wallet flow when a session exists. This is an afternoon of work and it unblocks everything.
2. The Borrow API. Teach it the post-migration collateral layout so deposit and borrow can build transactions again, make the button say why when it cannot, and compute the health factor the way the docs define it.
3. Order lifecycle. Check the wallet's network before building an order, show the real error text, and give every order a visible state: submitted, awaiting signature, pending fill, filled, failed with reason. Make Cancel real, refunding from the escrow before the sweep, or remove the button. Users should never learn from an empty portfolio that nothing happened.
4. Price integrity. Put AAPL in the batch request, show the oracle price and its age on the confirm button, and refuse an order when the panel's price and the oracle diverge beyond a tolerance. This is cheap, and it is the first thing a sceptic checks.
5. One source of truth. Tour, docs, chatbot and app state should be generated from the same place. Remove the tranche steps from the tour until Earn ships.
6. Cost transparency at the point of decision. The oracle price the order will use and its age, the fee or haircut actually taken at fill, the exact share quantity, what happens when the market is closed, the estimated assignment cost with a sentence on what it means, and a health-factor preview before a leveraged buy.
7. Proof of reserves as a link, not a footer label. Right now "Onchain proof of reserves" is plain text with nowhere to click.
8. Gate the read endpoints that return positions and balances by address.
9. Mobile parity for the order panel.

## The economics, read as a sceptic

The pitch is the volatility risk premium: implied volatility on equity options has, on average and over long periods, exceeded realised volatility, so systematic call writers collect more than they pay out. That is well documented and I do not dispute it. Spout's twist is to route the premium to stablecoin lenders and give the share holder a free loan in exchange for capping their upside. The premium exists. The question is who pays in the bad weeks, and here the documents disagree with each other.

### Who bears assignment

The settlement page says the engine computes "premium received minus the cost of assignment" for assigned positions, and the loss-waterfall page pushes any net cycle loss through the insurance fund, then the junior tranche, then senior. That reads as the protocol making assigned borrowers whole. The assignment page and the "Carol" scenario say the opposite: assigned shares are sold at the strike, auto-roll rebuys at the higher market price, and Carol "ends the cycle with slightly fewer shares". The site's own FAQ is blunter still: "If an option expires in the money, you absorb the difference between the strike price and the market price for that cycle. Historically this cost averages around 0.5% annualized." Both models cannot be true at once. If borrowers lose shares on assignment, then "0% interest" carries a path-dependent cost that shows up exactly in the weeks the stock does best, which is the week a leveraged holder least wants to be sold out. If the protocol absorbs it, the junior tranche is far more exposed than a 32% target implies.

The app's per-asset "Est. Borrow cost" column is the FAQ's 0.5% made per-asset. The public instruments endpoint carries it as a field, and the values are 0 basis points for NVDA, SMCI and BSOL, 6 for GLD, 7 for PFE, 54 for GOOG, 55 for IBIT, 58 for MSTR, 80 for XOM and 86 for GS. Assignment cost rises with volatility, so the three most volatile names at zero and Goldman at the top is not a model output. Until it is, the column should say "estimate pending" rather than 0.00%.

### Pooling

Writing one set of calls per asset against the aggregate is what makes sub-100-share participation possible, and it smooths outcomes. It also means a borrower who never wanted to sell gets assigned pro rata because someone else in the pool wanted maximum leverage. That is a fair trade for a free loan; it needs to be said plainly at the point of borrowing.

### The lender math, with the collateral base stated

The docs' worked example collects $30,000 of gross premium in a week on a $10m lending pool, which is 0.3% of the pool per week or roughly 15.6% annualised before the 20% protocol fee. After the fee, $24,000. Senior's 7% priority on $8.5m is about $11,400 a week, leaving $12,600 to split 25/75, so senior ends near 8.9% and junior near 32.8%. The arithmetic checks.

The premium, though, is not written on the lending pool. It is written on the collateral, and at a flat 50% LTV a fully lent $10m pool sits against at least $20m of locked shares. So the basket only has to yield about 0.15% a week, roughly 7.8% a year, in out-of-the-money call premium. That is achievable for MSTR, SMCI, NVDA or IBIT and thinner for AAPL, PFE, GLD or XOM at strikes "well above the current price", so the blended yield depends on how much of the collateral sits in the volatile names, and the junior return is the residual of that mix. Two things the example leaves out matter more than volatility. Utilisation: capital that is not borrowed earns only the money-market base yield, so senior's 7% priority depends on how much of the pool is actually lent, and the settlement page admits the priority can go unpaid in some weeks. And the split: the example hands 100% of net premium to lenders, while three other pages say a locked borrower earns "the borrower's share of cycle yield" and the distribution page also deducts an "LP reserve" before lenders are paid. Read the 8.9% and 32.8% as upper bounds from a page that ignores its neighbours.

### The 50% LTV, the health factor and what a liquidation costs

The borrow side is simpler and better specified. Health factor is collateral value times the asset's liquidation threshold, divided by debt. NVDA's threshold is 58.8% LTV, so a maximum borrow at 50% opens at a health factor of about 1.18 with an 8.8% cushion. When it hits 1.00 the protocol sells just enough collateral to restore it, and the liquidation fee equals the asset's buffer, 4% on the steadiest names up to 12.5% on the most volatile, charged on the slice sold. In the docs' own example a 15% drop in NVDA costs 21 of 100 shares. A flat LTV across eleven assets is easy to audit and hard to game, so I like it. The unpriced risk is the gap: if liquidations run only during US market hours, which the chatbot says and the docs leave open, a weekend move larger than the cushion is bad debt for the pool, and IBIT and BSOL track assets that trade all weekend. The buffer per asset should be shown in the app next to the health factor, and the market-hours rule should be documented rather than left to a chatbot.

### Buffers

The insurance fund targets 2% of pool value, which is about seven weeks of gross premium in the example. Junior is 15% of the pool. A single-name assignment loss on a 15% rally past an 8% strike costs roughly 7% of that name's notional. With 40% of the collateral in that name it is 2.8% of collateral, which at 50% LTV is about 5.6% of the $10m lending pool before netting the week's premium. Fund plus junior absorb it and senior is untouched. A broad market melt-up across the basket is the scenario that reaches senior, and the docs' answer is "has not occurred in any historical scenario we have tested". I would want the backtest published, with the basket weights and the strike rule, before believing the senior tranche is as safe as the marketing implies. The waterfall itself is described two ways: the loss-waterfall page says insurance fund, then junior, then senior; the "who this is for" page says insurance fund, then treasury, then pool socialisation. Pick one.

### Liquidation hours and the oracle

Docs say health-factor checks continue off-hours at reduced frequency and leave open whether a liquidation can execute then. The chatbot says liquidations happen only during US market hours and that buffers absorb overnight and weekend gaps, with higher buffers for IBIT and BSOL. Tokenized shares trade on chain 24/7; the underlying does not. The oracle keeps updating through the night (the devnet feed was being refreshed every few minutes at 18:00 WIB with the US market closed), so a price exists at all times, which makes the question of when liquidations can run more important, not less. The market-hours-only answer is the honest one, because a liquidation needs a real sale at the broker. It should be the documented one, and enforced. On the order side the same rule is keeper habit rather than program law: the chain shows one buy filled at 07:33 UTC on a Wednesday, six hours before the open, and nothing on chain would have stopped it.

### Exits

Senior gets instant withdrawal from a reserve targeting 15% of the pool with a 0-3% haircut that rises as the reserve drains, then a FIFO queue, then a market for queued claims. Junior gives 45 days' notice and keeps bearing losses while it waits. This is a well-designed run brake. It also means "no lockup on senior deposits" is true only while the reserve holds. The accurate sentence is "instant up to the reserve, queued after".

### Compliance, as seen on chain

The NVDA spAsset mint on devnet is a Token-2022 mint with default-frozen accounts, a permanent delegate and a metadata pointer. A new holder's account starts frozen until the issuer thaws it after KYC, which the fill instruction does ("freeze gated" is in its name), and the issuer can move or burn any holder's tokens at any time. That is the standard pattern for regulated security tokens and it is defensible. The docs describe a transfer hook instead, and nowhere does the product tell the user that a permanent delegate exists. The token program itself does: when the keeper created my NVDA account it logged "Mint has a permanent delegate, so tokens in this account may be seized at any time". A user should read that sentence before the first buy, in the app, not in a block explorer. The docs also describe a "Path B" where you deposit xStocks or Ondo tokens without KYC through Spout; there is no such flow in the app. On devnet the identity account that marks a wallet as verified is created automatically, and the program logs say so: "DEVNET: KYC verification bypassed (mock)".

### Custody, keys and status

The app names Alpaca Securities as the custodian and shows reserves at 100.2%; the site says FINRA-registered and SIPC-covered. Spout's own registration is as a FinCEN money services business, which covers AML obligations and is not broker-dealer status. Offering tokenized US equities and margin-like credit to non-US persons is the regulatory question that decides whether this product exists in two years, and I am not the person to answer it.

What I can describe is how the devnet deployment is operated, because it is all on chain. None of the Spout programs publish an IDL. The orders program and the vault program share one upgrade-authority key, and that same key fills orders, pushes the Stork price updates to the oracle program, and signed the transaction that created my KYC identity account. A second key is the NVDA token's mint authority, permanent delegate and metadata authority at once. One hot key that can set the price, settle against it, grant KYC and upgrade the program is fine for a devnet. On a mainnet that custodies collateral it will not survive an audit. "Onchain proof of reserves" in the footer is plain text with no link; I could not find an attestation account to check.

The USDC a buyer escrows does not stay in escrow. A plain token transfer signed by a third off-chain key sweeps it to a treasury wallet about two and a half minutes after every buy, fill or no fill, and that wallet held about 42,000 USDC when I looked. Sells are paid out of a separate pool of about 97,000 USDC owned by the vault. On devnet these are test dollars. On mainnet the same design means a pending order is an unsecured claim on the operator until the keeper fills it.

### Taxes

Non-US holders lose 30% of dividends to withholding at the broker; premium income is not withheld. The year-end summary in Settings is the right idea, and the "BC" column in it needs a name.

## Before I put real shares in

Reload the page and still be connected. Place an order and see it fill, with the price on the button before I sign and the fee I paid after. A cancel that works. A sentence on the buy screen about assignment: how often, what it costs me, and whether I end up with fewer shares. The permanent delegate disclosed where I sign. Proof of reserves I can click. The lender backtest with basket weights. Verified programs with a multisig or timelock on upgrades, and the oracle relay, settlement and upgrade roles on separate keys.

## Appendix

- Tested 10 September 2026, 17:00 to 18:30 WIB and 18:45 to 21:00 WIB; the US market opened at 20:30 WIB. Build v1.0.0 (5d92f4c).
- My transactions on devnet: buy placed 4yiwJn3XY7ptPH9e7X224zPmuy3tFnrC4RXSF8tLrcUU6DCaPXzd4fd8e6pMVFwMbnMY9LkYiGEM51dJ684S5Q7U (12:35:00 UTC, 10 USDC for 0.045124317 NVDA at 221.61, oracle staleness 190 s); escrow swept 4sNA49qVpkv4zGSVgzKgQjZtpYhsFZSoWeEkibZr13YFPrvnTmLy9DxdX1ahjAruvd6ErqfkRqCFaCnhdSyiQBrX (12:37:30); cancel requested through the app at 12:36:50 with no on-chain transaction; filled 4WwezzzT5ShBtdKBMD2fqWhqzhy3i74hFuCPwDzAptctFm1GDkfJD8zr6anhyUmS7XHrgQjenNYJYK3X7snAFTzN (13:30:58, 0.045302013 NVDA minted); sell placed 4i55d8NZmu5C3iLGsnP5nFAMptxoG3wD5GqAMzRuezBFunnCaDQasve4XjFDk4qnd7tUGYP81p1PgBffVT6Rq5uQ (13:40:16, 0.0182807 NVDA at 218.81) and filled 2mVKyNRkY6bdjVqVQF56KBEf578VVfotqpnCP5DeCdQDffiKFETaW8Ua8aCRj8Zrzct2RdjxmNuYht9vo17W4pBb (13:40:25, 3.998976 USDC paid).
- Test wallet LKjgX4UBCy7bLi8tJBbW4oE2Azu4hbiR8t19VJYnTk7 on devnet. Circle devnet USDC mint 4zMMC9srt5Ri5X14GAgXhaHii3GnPAEERYPJgZJDncDU.
- App endpoints seen: /api/market-data/current-price, /api/market-data/bars, /api/market-data/instruments, /api/orders/buy, /api/orders/submit, /api/orders/cancel, /api/orders, /api/spout-orders/{id}/status, /api/vault/positions, /api/vault/deposit, /api/vault/borrow, /api/wallet/balances, /api/users/transactions. Auth via auth.privy.io. RPC api.devnet.solana.com.
- Programs in the buy transaction: SPoRXgsB4gWZmWPwyndoRWQrmZXKUc7o7oPMdkkGRcG (orders, upgrade authority 7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp), SKYCVrkX3mQaHwZcLrUtuvzii43kma7kBUim5MaQm6k (identity, authority BD29wQ5Tj7b1MEFqquRxASsTqoN38oCrjxpU4riTZx7C), plus Token, Token-2022 and System. Other Spout programs on devnet: spvaDgABYdpFKyatqo4Jr3nfwvVgWF5BbFxKFkzN3Am (vault, same authority 7N31cE8B…, not exercised by me). routARWktuwebKa7YrWrqJrCYsinGRFzsTuzHtPwgfn has a different upgrade authority, Ho7bjZdiDazBh1DjB8XuKVWmxvhf8YwYBD3CdJJeFM3M, which is also the key that deployed and upgraded the orders program 35 times (last on 21 August), so it is Spout's. Oracle program stork1JUZMKYgjNagHiK2KdMmb42iTnYe9bYUCDUk8n. NVDA spAsset mint FkzjAJX584L1LncQb8AKSnGhc5sLvZjPGCgZLGpeHyTH (9 decimals; mint authority, permanent delegate and metadata authority 28bLmwghfAyZLCXpRZHpxa7uefN1TupnCftXfTqHzVJm; freeze authority GDBgF4AdFnK3ku8e7euZZLUag3JU9zDzbGktXTrzfMtE).
- Keeper, oracle relay and identity-creator key 7N31cE8BRTpyAVDeczkutP4EnGdQJLSQm4q3SJ6aYWEp. Escrow USDC account ADZRYU7F5t4vzhBCihr9faNEYPbGAHYCiC8c3mkZKR2d, swept by 7YBM7UQitFS9rJR2YEfnXmGEJREfstKfRBjZRRyatqo2 to treasury token account Cr2A3ck62Fh4VQZ87UPMcxeW7zhEZBR5cxzMtNWtKEuz (owner Ee7LLsoSAYfs1XqUgtyLZqosXTS9keoTBAqsv3vfEpAd). Sell payout pool 5fcfJCZu2SskQCTm78LKS6dnww5quMR3cffG9HK2VkmE. Stork price feed accounts: NVDA 9E1ypNM4CsSdSmgLpi46XDea9Nv7Yrh7GSrxA98nyTsA, AAPL 3PD8BMMTDsgg5CaQ4mMsc8MnSWPGKk3aAj1ZDvUvfy7g (read $318.198 at 12:04 UTC on 10 September). AAPL spAsset mint B3S7TuCmBPsNJW32SzRLBwbmWmgNzFBuHnBG9rdvt6K5. Orders program history read for 7 to 10 September: 252 transactions, 117 orders, 135 fills, 0 failures; example AAPL fill 2nd8sUh4zmttozokZK7iPhSM1g5hBQLB9Hj9Uggjws2eJisUNJFBKnqxQLxSDBGj5vbQVUjfRSLNjKUE9XJn6kdN. Docs and site pages read on 10 September 2026.
- Not tested: phone widths on a real device, the leverage slider by hand, borrow, repay, liquidation and lending (Borrow API down, Earn not live). Other wallets' fills and the whole cancel history were read from the chain, not exercised.

I have no position in Spout and was not paid by them. This was written for the Superteam Earn beta intelligence challenge.
