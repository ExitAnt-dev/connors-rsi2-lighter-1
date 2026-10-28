# ExitAnt Connors RSI2 — Lighter

Deploy this automated trading bot to **your own Render account** as a background worker. The bot runs from a prebuilt Docker image;
this repository contains **only the Render Blueprint (`render.yaml`)** — no source code. One repository per image tag (`connors-rsi2-lighter-1` … `-6`).

The bot trades **Larry Connors RSI2** or **Turtle + Connors RSI2 merge** on [Lighter](https://app.lighter.xyz) USDC perpetual futures with exactly the
rules of the TanQuant backtest engine, on the exchange's own candles — daily (the original rule), 4-hour or 1-hour, chosen in the bot; UTC-aligned,
stocks/ETFs only US trading days and, on 4-hour/1-hour, only NYSE regular-session candles — for every indicator. On daily candles, Yahoo daily
candles are used only to complete the 200-day moving average of symbols with fewer than 200 exchange candles (recently listed).
Copy a TanQuant result with **“Copy bot setup”** and paste it into the bot to set everything up at once.

## Deploy to Render

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-lighter-1)

```
https://render.com/deploy?repo=https://github.com/ExitAnt-dev/connors-rsi2-lighter-1
```

1. Click the button (or open the link) and sign in to Render.
2. Render reads `render.yaml` and proposes **one background worker** in the **Frankfurt** region with a **1 GB persistent disk** mounted at `/data`
   (settings, books, positions and the bot's own API key survive restarts and redeploys). Keep the region: Lighter refuses every write
   (orders, key registration) from US addresses — “restricted jurisdiction” — so a US region (Oregon, Ohio, Virginia) cannot trade.
3. Enter the environment variables listed below. They are marked `sync: false`, so Render asks you for them on the deploy screen
   and stores them **only in your own Render account**.
4. Click **Apply** and wait for the first deploy to finish.
5. Open Telegram and send **`/start`** to your bot. The bot registers its trading key and approves the ExitAnt fee by itself at the first start
   (see below) — nothing to sign or paste.

## Before you deploy

- **A Lighter account, on a wallet used only for this bot.** Create a new MetaMask account for the bot, open [app.lighter.xyz](https://app.lighter.xyz),
  connect that wallet and deposit USDC — the first deposit creates the Lighter account. An existing Lighter account works too, but the bot needs the
  wallet's private key, so keep only the funds you trade on that wallet — not your main wallet.
- **The private key of that wallet** (MetaMask → account menu → *Account details* → *Show private key*; 64 hexadecimal characters). This is the only
  credential the bot needs. It uses the key for two signatures only — registering its own trading API key and approving the ExitAnt fee — never for
  withdrawals or transfers out of the account, and it never writes the key to logs or files. The key sits only in your Render environment variable,
  so keep your Render account secure (2FA). **The bot never asks for a key in Telegram** — a message asking for one is not from the bot; if you paste
  a key into the chat by mistake, the bot deletes the message and warns you (treat that key as exposed and move the funds to a new wallet). Do not create an API key in the Lighter app for the bot; the bot makes and registers its own
  (key index 250) at the first start.
- **USDC in the Lighter Perps account** (deposits go to Perps by default; USDC on Spot is not collateral). The trading key cannot be registered
  while the account holds no collateral.
- **Simple Trading Account** (the default account type; the API calls it "classic"). If the account is a Unified Trading Account, the bot switches it
  to Simple at the start (turning off any non-USDC asset used as margin first). It does not switch while a position is open — close the positions,
  or switch it yourself on the Lighter web app (app.lighter.xyz; the account type can be changed only on the web). Open spot orders block the change.
- **The ExitAnt fee approval** happens automatically at the first start: the bot attaches ExitAnt's integrator fee (0.1% taker/maker) to its orders,
  and Lighter requires a one-time wallet-signed approval of that fee, which the bot signs with the wallet key. It is done once per account and
  never repeated. Without a confirmed approval nothing is traded.
- **Plus account tier.** The fee can only be charged on a Plus (or Premium) account. If your account is Standard, the bot asks for your consent
  in Telegram and switches it to Plus for you; Plus adds Lighter's own 0.005% maker/taker fee to every trade (Standard is 0%).
  You can switch back in the Lighter app after 24 hours.
- Lighter holds **one position per symbol**: a symbol runs long *or* short (the “Both” direction is not offered, also when changing settings).

## First start

Right after the 🚀 welcome message the bot finds your Lighter account from the wallet address, registers its trading key (a few seconds),
approves the ExitAnt fee with the wallet signature and reports “✅ ExitAnt Membership: 🟢 Verified”. If the account is on the Standard tier it
asks once for your consent to switch to Plus. Then send `/start`, choose the language, accept the terms and paste your TanQuant setup (or go
through the wizard), and press ▶️ Start.

If the bot reports “no Lighter account for this wallet”, the wallet has not deposited on Lighter yet — deposit through the invite link above.
If it reports that the key cannot be registered because the account holds no collateral, deposit USDC to the Perps account and restart.

## Environment variables

| Variable | What to enter |
|---|---|
| `LIGHTER_WALLET_KEY` | The private key of the MetaMask wallet that created the Lighter account (64 hexadecimal characters; a leading `0x` is optional) |
| `TELEGRAM_BOT_TOKEN` | Bot token issued by [@BotFather](https://t.me/BotFather) |
| `TELEGRAM_CHAT_ID` | Your own Telegram chat ID — the bot ignores every other sender |

Nothing else to configure: the exchange, trading mode and state-file location are built into the image.

## What the bot does

- Decides once per candle of the chosen length — daily at UTC midnight, 4-hour/1-hour right after each candle closes (stocks: session candles only) —
  and sends market orders at once (the backtest's “fill at the close”). On 4-hour/1-hour candles every indicator length is counted in candles, as in TanQuant
  (200-day MA → 200-bar MA).
- In the Turtle + Connors merge the Turtle side can run on its own candle length (e.g. Turtle 4-hour, Connors 1-hour) — chosen on the first Turtle
  screen of the wizard and under 🐢 Turtle rules in settings. Keeping both the same is the TanQuant backtest setup; a different pair is a bot-only
  setting that TanQuant cannot backtest (the Turtle channel and its ATR stop come from the Turtle candles, each side decides at its own candle close).
- Margin-saving entry (bot-only, not in the TanQuant backtest): when a new lot enters while the symbol already holds another lot (Turtle long,
  Connors long or Connors short — same direction or not), it uses the set leverage × 2 and ½ of the held lot's entry size; if the exchange still
  refuses for insufficient margin, × 4 and ¼ once more. The leverage goes back to the setting when that lot closes. The wizard, the settings summary,
  the positions screen and the entry notice explain it.
- Places an exchange stop-loss order per symbol and direction (the farthest stop of the open lots) and watches the closer stops and the exit rules itself
  around the clock — **keep the bot running**.
- Keeps one account per symbol (trading pocket + vault), like TanQuant's multi-asset backtest. The Turtle merge moves part of the profit to the
  vault (your account's **Spot USDC**) on a trend exit and reinjects it after a deep drawdown, by the USA Turtle rules.
- Every order carries ExitAnt's integrator attribution (0.1% taker/maker), credited to ExitAnt on Lighter.
- Re-checks every hour while trading that the account still meets the conditions (fee approval, Plus tier). If the approval is missing it renews it
  with the wallet signature; if the tier no longer holds, it stops opening new positions — the stops and exits of held positions keep running — and
  asks you in Telegram.
- Never stops trading because of an exchange answer — a refused or failed request skips that symbol for that cycle and is retried.

⚠️ Futures trading carries a very high risk of loss. Backtest results do not guarantee future returns.
