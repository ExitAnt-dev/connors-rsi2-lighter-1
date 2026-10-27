# ExitAnt Connors RSI2 — Lighter

Deploy this automated trading bot to **your own Render account** as a background worker. The bot runs from a prebuilt Docker image;
this repository contains **only the Render Blueprint (`render.yaml`)** — no source code. One repository per image tag (`connors-rsi2-lighter-1` … `-6`).

The bot trades **Larry Connors RSI2** or **Turtle + Connors RSI2 merge** on [Lighter](https://app.lighter.xyz) USDC perpetual futures with exactly the
rules of the TanQuant backtest engine, on the exchange's own daily candles (UTC day; stocks/ETFs: US trading days only) for every indicator.
Yahoo daily candles are used only to complete the 200-day moving average of symbols with fewer than 200 exchange candles (recently listed).
Copy a TanQuant result with **“Copy bot setup”** and paste it into the bot to set everything up at once.

## Deploy to Render

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-lighter-1)

```
https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-lighter-1
```

1. Click the button (or open the link) and sign in to Render.
2. Render reads `render.yaml` and proposes **one background worker** with a **1 GB persistent disk** mounted at `/data`
   (settings, books and positions survive restarts and redeploys).
3. Enter the environment variables listed below. They are marked `sync: false`, so Render asks you for them on the deploy screen
   and stores them **only in your own Render account**.
4. Click **Apply** and wait for the first deploy to finish.
5. Open Telegram and send **`/start`** to your bot. Right after the welcome message the bot asks you to **approve the ExitAnt fee once**
   (see below) — trading does not start before that.

## Before you deploy

- **A Lighter account created with ExitAnt's referral code `764313PR`.** Open [app.lighter.xyz/?referral=764313PR](https://app.lighter.xyz/?referral=764313PR)
  with a wallet (MetaMask) that has **never been used on Lighter**, make sure the referral code shown at sign-up is `764313PR` (enter it if the
  field is empty), connect and deposit USDC — the first deposit creates the account with that code. Lighter sets the code only when the account
  is created and it can never be added later, so an existing Lighter account cannot be used — create a new wallet account. The bot checks the
  code at every start and once an hour while trading.
- **An API key** from [app.lighter.xyz/apikeys](https://app.lighter.xyz/apikeys): connect the wallet, click *Generate API Key* and copy the three
  values the page shows — **Your Account Index**, **API Key Index** and **Private Key**. Leave **Maker Only off** (a maker-only key cannot place
  the bot's market and stop orders). Do not delete that key while the bot runs.
- **USDC in the Lighter Perps account** (deposits go to Perps by default; USDC on Spot is not collateral).
- **Classic account mode** (the default). In Unified mode the bot does not start — switch it in the Lighter app settings while no position is open.
- **The ExitAnt fee approval (once).** The bot attaches ExitAnt's integrator fee (0.1% taker/maker) to its orders; Lighter requires the account
  owner's wallet to approve that once with a signature. The bot cannot sign for you (it has no wallet key): after the welcome message it shows a
  text — copy it, sign it with the wallet that owns the account (MetaMask “Sign Message”, e.g. through the signing tool linked in the chat)
  and paste the resulting signature into the chat. The bot sends the approval and continues. Without the approval nothing is traded.
- **Plus account tier.** The fee can only be charged on a Plus (or Premium) account. If your account is Standard, the bot asks for your consent
  in Telegram and switches it to Plus for you; Plus adds Lighter's own 0.005% maker/taker fee to every trade (Standard is 0%).
  You can switch back in the Lighter app after 24 hours.
- Lighter holds **one position per symbol**: a symbol runs long *or* short (the “Both” direction is not offered, also when changing settings).

## Environment variables

| Variable | What to enter |
|---|---|
| `LIGHTER_API_KEY` | The API key's **Private Key** (`0x` + 80 hex characters) from the Lighter API key screen |
| `LIGHTER_ACCOUNT_INDEX` | **Your Account Index** from the same screen |
| `LIGHTER_API_KEY_INDEX` | **API Key Index** from the same screen (4–254) |
| `TELEGRAM_BOT_TOKEN` | Bot token issued by [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_CHAT_ID` | Your own Telegram chat ID — the bot ignores every other sender |

Nothing else to configure: the exchange, trading mode and state-file location are built into the image. The bot never sees your wallet key.

## What the bot does

- Decides once per daily candle, right after the exchange closes it (UTC midnight), and sends market orders at once (the backtest's “fill at the close”).
- Places an exchange stop-loss order per symbol and direction (the farthest stop of the open lots) and watches the closer stops and the exit rules itself
  around the clock — **keep the bot running**.
- Keeps one account per symbol (trading pocket + vault), like TanQuant's multi-asset backtest. The Turtle merge moves part of the profit to the
  vault (your account's **Spot USDC**) on a trend exit and reinjects it after a deep drawdown, by the USA Turtle rules.
- Every order carries ExitAnt's integrator attribution (0.1% taker/maker), credited to ExitAnt on Lighter.
- Re-checks every hour while trading that the account still meets the conditions (ExitAnt referral code, fee approval, Plus tier). If one no
  longer holds, it stops opening new positions — the stops and exits of held positions keep running — and asks you in Telegram to complete that step again.
- Never stops trading because of an exchange answer — a refused or failed request skips that symbol for that cycle and is retried.

⚠️ Futures trading carries a very high risk of loss. Backtest results do not guarantee future returns.
