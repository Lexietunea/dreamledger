# DreamLedger

**A rule-guarded perps bot for dreamDEX that writes every trade, and every trade it refuses, to an onchain journal on Somnia.**

*DreamLedger is an independent community project. It is not affiliated with or endorsed by dreamDEX or Somnia.*

Built for the [dreamDEX Perps Builder Bounty](https://dorahacks.io/hackathon/dreamdex-perps-bounty/detail) (MVP track, *bots and agents*). Runs on the **Somnia Tokyo testnet** (Hideki, chain ID `50383`). Testnet only: no real money.

> This repository is a fork of the official [dreamdex-bot-kit](https://github.com/somnia-chain/dreamdex-bot-kit) (MIT). The original kit README is kept at [`docs/KIT-README.md`](docs/KIT-README.md). Everything new lives in two folders:
> - [`strategies/perp-journal`](strategies/perp-journal): the bot
> - [`journal-contract`](journal-contract): the `DecisionJournal` smart contract

## What it does

You describe a strategy in a small `rules.json` file: the market, long or short, position size, leverage, take-profit, stop-loss, and a set of safety limits. The bot then:

1. Publishes your rule set to the journal contract (its hash stamps every later entry).
2. Waits for an entry signal (`manual`, `breakout` or `meanReversion`).
3. Runs **pre-trade checks** before every order: data freshness, daily loss limit, open positions, cooldown, spread, leverage, position size and risk per trade.
4. If a check fails, it **refuses the trade** and writes `REFUSED` plus a reason code (for example `MAX_SPREAD`) to the journal.
5. If all checks pass, it places a real perps order on dreamDEX and writes a `TRADE` entry.
6. It watches your take-profit and stop-loss and, when one fires, closes the position and journals the close.

Anyone can read the journal and check that the bot did only what the published rules allow, including every time it said no.

## Who it is for

- **Traders who want automation without a black box.** The rules are a short file, the limits are enforced on every order, and the record cannot be edited afterwards.
- **People who want to show what their bot did.** A public, onchain log of decisions is checkable proof.
- **Builders** who want a small, readable example of guardrails plus an onchain audit trail on top of the dreamDEX bot kit.

## How it works with dreamDEX

```
rules.json ──> perp-journal bot ──> dreamDEX perps (Tokyo testnet): real orders
                    │
                    └──> DecisionJournal contract (Somnia Tokyo): trades + refusals
```

- The bot uses a **trading key** linked to your dreamDEX account through *Link a bot* in the app. It can place and cancel orders for your account but **cannot withdraw or send funds**.
- Orders go through the kit's `perp-core` library, the same path as the kit's `perp-starter`.
- The journal is a separate contract written to with the same bot wallet.

## Proof on Somnia Tokyo testnet (chain 50383)

**Journal contract:** `0x259D6dEaa3027ccFda8ff9b088AC9d8048F656a9`

| What | Reference |
|---|---|
| Rule set published (dry run, ETH-PERP) | tx `0x8e9166b559cb95f5ccffc187cb241db122f3c7946f75bf7cff2c75b87e065425` |
| **Refusal journaled:** `REFUSED MAX_SPREAD` (spread 2.9 bps vs a 1 bps limit) | tx `0x450683639b32126bc3b933fb8a99b89a67deb11163c266db847331ce864f038f` |
| Rule set published (BTC-PERP, guarded trade) | tx `0x6ec1397356bc6b110819cc2b7af17ed5f7ce44fc361c6b8c8647c614af77edb4` |
| **Guarded trade:** long BTC-PERP via `perp-journal`, 0.00066 BTC @ 76030.9 | order `9573860174255268254693`, tx `0xa337185162df5996c477879b5ba1e16e026aa8d4aa3c1f26ce4ebfedb4d706b1` |
| Trade journaled: `TRADE open` | tx `0xfc9112a01f3e8ba13ce0a6c101daabbbcf31c0c1471c15ef72b74cff908e36f0` |
| Earlier real order (kit `perp-starter`, long BTC-PERP, Oct 28) | order `387381625547908994209`, tx `0x854ff9b53193b1cebf32fab32c93b36ded3b61adae4640f98fdf26656c4036ca` |
| Earlier real order (kit `perp-starter`, long BTC-PERP, Nov 2) | order `8116567392432212187748`, tx `0x625bc3977b1211fd83553307b06ef83e7e566d37b8d1bdd683e62608cbc5d390` |

## Run it

**Prerequisites:** Node.js 22+, a dreamDEX testnet account at [app.testnet.dreamdex.io](https://app.testnet.dreamdex.io), and a **new testnet-only wallet** for the bot. Never use a wallet that holds real funds.

1. **Install.**
   ```bash
   git clone https://github.com/Lexietunea/dreamledger.git
   cd dreamledger
   npm install
   ```
2. **Link the bot.** In the app, open your wallet, choose **Link a bot**, paste the bot wallet's address, and use **Top up** to give it gas (STT). Make sure your account has perps collateral.
3. **Deploy the journal** (one time).
   ```bash
   cd journal-contract
   npm install
   cp .env.example .env        # set DEPLOYER_PRIVATE_KEY to the bot key
   npm run deploy              # prints the contract address
   cd ..
   ```
4. **Configure the bot.** Copy `.env.example` to `.env` in the repo root and set:
   ```
   PRIVATE_KEY=<bot key>
   OWNER_ADDRESS=<your dreamDEX account address>
   NETWORK=hideki
   RPC_URL=https://api.hideki2.infra.testnet.somnia.network
   JOURNAL_ADDRESS=<address from step 3>
   DRY_RUN=true
   ```
   Check the setup with `npm run perp:doctor`.
5. **Edit your rules** in `strategies/perp-journal/rules.json`.
6. **Dry run** (no orders sent; refusals can still be journaled):
   ```bash
   JOURNAL_IN_DRY_RUN=true npm run dev -w perp-journal
   ```
   On PowerShell: `$env:JOURNAL_IN_DRY_RUN="true"; npm run dev -w perp-journal`.
7. **Real run:** set `DRY_RUN=false`, set the account's leverage in the app to match your rules, and run `npm run dev -w perp-journal`. Keep the process running: see limitations below.

To see a refusal quickly, set `"market": "ETH-PERP"` and `"maxSpreadBps": 1` in `rules.json` and do the dry run.

### Environment variables added by this project

| Variable | Meaning | Default |
|---|---|---|
| `JOURNAL_ADDRESS` | Deployed `DecisionJournal` contract | none (journal off) |
| `JOURNAL_IN_DRY_RUN` | Write journal entries even when `DRY_RUN=true` | `false` |
| `JOURNAL_REFUSAL_THROTTLE_MS` | Minimum time between journal writes for the same refusal reason | `300000` |
| `PERP_EQUITY_USDSO` | Equity used for sizing and risk checks | `3000` |
| `RULES_FILE` | Path to the rules file | `rules.json` |
| `PERP_TICK_MS` | Polling interval | `10000` |

### `rules.json`

```json
{
  "market": "BTC-PERP",
  "side": "long",
  "entry": { "type": "manual", "timeframe": "10s", "lookback": 20 },
  "sizing": { "riskPctPerTrade": 2, "maxLeverage": 2, "maxPositionUsd": 50 },
  "exits": { "stopLossPct": 0.3, "takeProfitPct": 0.3 },
  "limits": {
    "maxOpenPositions": 1, "cooldownMinutes": 5, "maxSpreadBps": 5,
    "maxDailyLossPct": 5, "maxDataAgeSec": 60
  }
}
```

Rule sets are validated at startup and rejected if they exceed hard caps (leverage above 10x, risk above 5% per trade, daily loss above 20%).

### Refusal reason codes

`MAX_LEVERAGE`, `MAX_SIZE`, `MAX_OPEN_POSITIONS`, `COOLDOWN`, `MAX_SPREAD`, `RISK_PER_TRADE`, `DAILY_LOSS`, `STALE_DATA`, `OTHER`.

## The journal contract

[`journal-contract/contracts/DecisionJournal.sol`](journal-contract/contracts/DecisionJournal.sol)

- `publishRules(json)`: emits the rule set and stores its hash.
- `log(action, side, reason, market, ref, note)`: appends a `TRADE` or `REFUSED` entry stamped with the active rule hash. A trade must carry no reason and a refusal must carry one.
- `getEntries(offset, limit)` and `entryCount()`: paginated reads. Each write also emits an `EntryLogged` event.
- Only authorized writers (the deployer and any address it adds) can write.

Tests: `cd journal-contract && npm test` (six tests covering ordering, reason rules, access control and pagination).

## Limitations

- **Exits only work while the bot runs.** With a trading key, the take-profit and stop-loss are watched by the bot, not armed on the exchange. If the process stops, the position stays open.
- **Exit levels are set from the live mark** when the bot starts watching, not from the fill price.
- **Equity is configured** with `PERP_EQUITY_USDSO`, not read from the account.
- **No AI parser yet.** Rules are written by hand; turning plain English into a rule set is planned.
- **Not yet used:** Somnia Reactivity. The bot polls every 10 seconds.
- Testnet only. This is not financial advice, and the checks reduce risk but do not make a strategy profitable.

## X posts

1. https://x.com/Ghost_Trader9/status/2119372462261473571
2. https://x.com/Ghost_Trader9/status/2120126459561873670
3. _add link_

## Credits

Built on the [dreamDEX bot kit](https://github.com/somnia-chain/dreamdex-bot-kit) (MIT, DreamDEX S.A.). The `perp-journal` strategy is derived from the kit's `perp-starter`.