# Delta-neutral bot for Robinhood-Lighter

[Русская версия](#русская-версия) | [English version](#english-version)

---

## English version

### Delta-neutral bot for Robinhood-Lighter

The bot splits accounts into clusters, picks a random coin from the config, has one account from the cluster place a limit order inside the spread, has the other accounts buy it up, holds for a while, then also closes and reassembles the cluster from other accounts

#### Detailed description

One cycle:
1. A random **cluster** of `clusterSize` accounts (1 maker + N−1 takers) and a market are chosen.
2. **Opening**:
   1. The coin's spread must fit `desiredSpreadTicks` — the number of FREE price levels strictly between the best bid and best ask:
      `round((ask−bid)/tick) − 1`. `0` = bid/ask on adjacent ticks (no gap). The number `N` is a minimum (there must be **at least** `N` free ticks).
   2. The maker places **one GTC limit order** for the whole size: if there's no spread and `desiredSpreadTicks` = 0 in the config, it's placed exactly at the current price with no improvement;
      if `desiredSpreadTicks` > 0, it improves the price by at least 1 tick.
   3. **1s pause, then re-read the top of the book.** If the maker is still best — the takers immediately place opposing GTC limit orders at the same price. If the maker got outbid —
      the bot checks what actually got filled: nothing filled → cancel the order and retry at a fresh price; partially/fully filled → the takers
      make up exactly the filled chunk with an aggressive IOC at the current price, while the
      uncovered remainder goes into the next attempt.
   4. **Reconciling actual positions** for each participant — if the self-match didn't come together fully,
      the shortfall is made up **at market**, after which the position is reconciled once more.
   5. If even the top-up doesn't square someone up — an emergency reopen of the whole cluster,
      the cycle is skipped.
3. **Hold** for a random amount of time from `holdTimeSec`.
4. **Close** (if `closePositions`): the same self-match mechanics, but
   if there's no spread right now — wait and recheck every 5s, the maximum
   time is set by `closeSpreadWaitSec`, after which it closes at market.
5. Pause for `loopDelaySec`, repeat.

#### Rounds, clusters, and account rotation

The bot works in **rounds**, so the cluster composition actually gets shuffled:

1. At the start of a round **all accounts are free** → the whole pool is split into clusters
   (`clusterSize`, e.g. `[2,4]`), **with no singles left over**: if 1 account remains after the split,
   it's merged into a cluster or the size is narrowed down to a pair.
2. Each cluster has **its own random hold** (`holdTimeSec`).
   The round ends when the **longest-held** cluster finishes. Shorter ones, once they finish a cycle,
   **repeat** it (new size/side) for as long as another full cycle still fits before the round ends.
3. The round waits for **all** clusters to close → all accounts become free →
   a **full reshuffle** and a new round.


#### Requirements

- Node.js 18+.
- accounts with a balance on Lighter
- proxies

#### Installation

```bash
npm install
``

after that

```bash
Copy-Item config.example.json config.json
Copy-Item pk.example.txt pk.txt
Copy-Item proxies.example.txt proxies.txt
```

or manually replace the files `config.example.json`, `pk.example.txt`, `proxies.example.txt` with their counterparts **without example**

Fill in `pk.txt` — account private keys, they must already be registered and funded, at least 2, and proxies in `proxies.txt`.
Configure `config.json`.

#### Launch

3. `npm run dev` or `npm run build && npm start`

### Referral code

This build of the bot only works if every account in `pk.txt` is registered
on Robinhood-Lighter under one of the codes `VLADEM`, `RHLIGHTER`,
`BULLRUNSOON`, `FREEDOMLIT`, `PERPLFCH`, `LIGHTERLFG`. Before start 
the bot checks each account's referral code if even one doesn't match any
accepted code (or has no code at all) the bot prints the list of 
non-matching accounts and exits

#### config.json

| Field | Meaning |
|---|---|
| `signerKeysFile` | additional keys are stored here — each wallet gets its own signing key generated on first run and saved here, so on restart the bot reuses the same key instead of registering a new one. **Don't send this to anyone** |
| `privateKeysFile` | path to the file the bot reads private keys from, default `./pk.txt`|
| `proxiesFile` |  path to the file the bot reads proxies from, default `./proxies.txt` |
| `markets` | lists the pairs to trade on (you can run `npm run info` to print the correct name of every pair), or set `"all"`, in which case a random suitable one is chosen |
| `clusterSize` | cluster size `[from, to]` — a cluster is several accounts where, e.g., 1 holds long and the rest hold short (or the other way around), clusters are reassembled every round|
| `rotationStateFile` | auxiliary file for the bot — stores how many times each account has been the maker, default `./rotation-state.json` |
| `signerKeysFile` | auxiliary file for the bot — stores accounts' API keys, default `./lighter-keys.json` |
| `firstSide` | can be set so long is always placed first, default is `RANDOM` |
| `positionSizeUsd` | position size to open, accounting for leverage, [from, to] — if `markets` is set to `all` then a suitable pair with enough leverage is searched for |
| `holdTimeSec` | hold time in seconds [from, to], better set to 30 minutes - 1 hour |
| `loopDelaySec` | pause in seconds before repeating, [from, to] |
| `marginSafetyPct` | percentage to shrink the position size by to avoid frequent margin-shortage errors, better left at 0.95  |
| `shrinkOnMarginFail` | useful setting — if the size is set to e.g. $1000, but the balance is only $20, the bot will gradually shrink the size until it fits the balance |
| `desiredSpreadTicks` | how much spread a pair needs to open a trade on it, measured in ticks — the smallest amount the price can be changed by, e.g. 1 cent for ETH|
| `takerDelaySec` | rarely used, for when a pair with a spread can't be found |
| `closeSpreadWaitSec` | how many seconds to wait for a spread to close with — only with it can you close at zero, it's usually found within 0-10 seconds, but I set 3600 for reliability |
| `closePositions` | `true/false` whether to use the same mechanics on close, or hold so it closes via TP/SL (the bot doesn't place these — set false only if you need it) if false, closes at market once the time is up | 
| `flattenOnStart` | `true/false` whether to close open positions, if any, on startup |
| `minBalanceUsd` | checks account balances on startup, if below this amount the account is simply ignored |
| `dryRun` | used for testing, if dryRun=true the bot doesn't make real trades but logs as if it were really working, default false |
| `telegramBotToken` | from [@BotFather](https://t.me/BotFather) |
| `telegramChatId` | your ID (get it from [@userinfobot](https://t.me/userinfobot)). |
| `telegramProxyUrl` | proxy for Telegram, so the bot can send messages from a server with any geo, if left blank it takes the first proxy from the `proxies.txt` file |

#### Security

- `pk.txt`, `config.json`, `proxies.txt`, `lighter-keys.json`,
  `rotation-state.json` — don't send these to anyone, they hold private keys and account data

## Русская версия

### Дельта нейтральный бот для Robinhood-Lighter

Софт делит аккаунты на кластеры, выбирает случайную монету из конфига, одним аккаунтом из кластера ставит лимитку между стаканом, выкупает её другими аккаунтами, держит некоторое время, также закрывает и пересобирает кластер из других аккаунтов

#### Подробное описание

Один цикл:
1. Выбирается случайный **кластер** из `clusterSize` аккаунтов (1 maker + N−1
   takers) и рынок.
2. **Открытие**:
   1. Спред монеты должен подходить под `desiredSpreadTicks` — количество
      СВОБОДНЫХ ценовых уровней строго между best bid и best ask:
      `round((ask−bid)/tick) − 1`. `0` = bid/ask на соседних тиках (зазора нет).
      Число `N` = минимум (свободных тиков должно быть **не меньше** `N`).
   2. Maker ставит **одну GTC-лимитку** на весь размер: если спреда нет и в конфиге
      стоит `desiredSpreadTicks` = 0, то ровно на текущую цену, без улучшения; 
      если `desiredSpreadTicks` > 0, то улучшает хотя бы на 1.
   3. **Пауза 1с и перечитывание топа книги**. Мейкер всё ещё лучший — тейкеры сразу
      встают встречными GTC-лимитками на ту же цену. Мейкера успели перебить —
      бот смотрит на факт исполнения: ничего не исполнилось → отменяет ордер и
      пробует заново на свежей цене; исполнилось частично/полностью → тейкеры
      добирают именно исполненный кусок агрессивным IOC по текущей цене, а
      непокрытый остаток идёт следующей попыткой.
   4. **Сверка по факту позиций** для каждого участника — если self-match
      собрался не полностью, недостача добирается **маркетом**, после чего
      позиция сверяется ещё раз.
   5. Если и доборка не выровняла кого-то — аварийное переоткрытие всего кластера,
      цикл пропускается.
3. **Холд** случайное время из `holdTimeSec`.
4. **Закрытие** (если `closePositions`): та же механика self-match'а, но
   Если спреда нет прямо сейчас — ждём и перепроверяем каждые 5с, максимальное 
   время задано в `closeSpreadWaitSec`, после этого закрывается по рынку.
5. Пауза `loopDelaySec`, повтор.

#### Раунды, кластеры и ротация аккаунтов

Бот работает **раундами**, чтобы состав кластеров реально перемешивался:

1. В начале раунда **все аккаунты свободны** → весь пул бьётся на кластеры
   (`clusterSize`, напр. `[2,4]`), **без одиночек**: если после разбивки остаётся
   1 аккаунт, он подклеивается к кластеру или размер сужается до пары.
2. У каждого кластера **свой случайный холд** (`holdTimeSec`).
   Конец раунда = когда закончит **самый долгий** кластер. Короткие, закончив цикл,
   **повторяют** его (новый размер/сторона), пока до конца раунда влезает ещё один
   полный цикл.
3. Раунд ждёт закрытия **всех** кластеров → все аккаунты освобождаются →
   **полная перетасовка** и новый раунд.


#### Требования

- Node.js 18+.
- аккаунты с балансом на лайтере
- парокси

#### Установка

```bash
npm install
``

после этого

```bash
Copy-Item config.example.json config.json
Copy-Item pk.example.txt pk.txt
Copy-Item proxies.example.txt proxies.txt
```

или руками заменить файлы `config.example.json`, `pk.example.txt`, `proxies.example.txt` на их аналоги **без example**

Заполните `pk.txt` - приватные ключи аккаунтов, они должны быть уже зарегистрированы и пополнены, минимум 2 и прокси в `proxies.txt`. 
Настройте `config.json`.

#### Запуск

3. `npm run dev` или `npm run build && npm start`

### Реферальный код

Эта раздача бота работает **только** если каждый аккаунт в `pk.txt`
зарегистрирован на Robinhood-Lighter по одному из кодов `VLADEM`,
`RHLIGHTER`, `BULLRUNSOON`, `FREEDOMLIT`, `PERPLFCH`, `LIGHTERLFG`. Перед стартом
 бот проверяет реферальный кодкаждого аккаунта
если хоть один не совпадает ни с одним из принятых кодов
(или не имеет кода вообще), бот печатает список несоответствующих аккаунтов
и завершает запуск

#### config.json

| Поле | Смысл |
|---|---|
| `signerKeysFile` | тут хранятся дополнительные ключи - каждому кошельку при первом запуске генерируется свой ключ подписи и сохраняется сюда, чтобы при рестарте бот переиспользовал тот же ключ вместо регистрации нового. **Никому не отправляйте** |
| `privateKeysFile` | путь до файла, с которого софт берёт приватники, по умолчанию `./pk.txt`|
| `proxiesFile` |  путь до файла, с которого софт берёт прокси, по умолчанию `./proxies.txt` |
| `markets` | тут перечисляются пары на которых торговать (можно прописать `npm run info` то выведется правильное название всех пар) либо написать `"all"`, тогда будет выбираться случайная из подходящих |
| `clusterSize` | размер кластера `[от, до]` кластер это несколько аккаунтов среди которых 1 держит например лонг а остальные шорт или наоборот, кластера пересобиратся каждый раунд|
| `rotationStateFile` | вспомогательный файл для софта, по этому пути хранится информация о том, какой аккаунт сколько раз был мейкером, по умолчанию `./rotation-state.json` |
| `signerKeysFile` | вспомогательный файл для софта, по этому пути хранятся апи ключи аккаунтов, по умолчанию `./lighter-keys.json` |
| `firstSide` | можно поставить чтобы всегда сначала ставился лонг, а по умолчанию `RANDOM` |
| `positionSizeUsd` | размер позиции на сколько открывать с учётом плеча, [от, до] если в `markets` стоит `all` то будет искаться подходящая пара у которой хватит плеча |
| `holdTimeSec` | время удержания в секундах [от, до], лучше ставить 30 минут - 1 час |
| `loopDelaySec` | пауза в секундах перед повтором, [от, до] |
| `marginSafetyPct` | процент занижения размера позиции чтобы избежать частые ошибки нехватки маржи, лучше оставить 0.95  |
| `shrinkOnMarginFail` | полезная настройка, если стоит размер например 1000$, а баланс только 20$, то софт будет понемногу понижать размер пока не хватит баланса |
| `desiredSpreadTicks` | сколько нужно спреда на паре чтобы открыть на ней сделку, меряется в тиках, это минимальная разница на которую можно изменить цену, например 1 центу у эфира|
| `takerDelaySec` | используется редко когда не получается найти пару со спредом |
| `closeSpreadWaitSec` | сколько секунд ждать спред для закрытия, только с ним получится закрыть в ноль, чаще всего находит за 0-10 секунд, но для надёжности ставлю 3600 |
| `closePositions` | `true/false` использовать ли такую же механику при закрытии или держать чтобы закрылось по тп/сл (софт не ставит, ставить false только если вам нужно) если false то закрывает по маркету по истечению времени | 
| `flattenOnStart` | `true/false` закрывать ли открытые позиции если они есть при запуске |
| `minBalanceUsd` | при запуске проверяет баланс аккаунтов, если он меньше этой суммы то просто игнорирует аккаунт |
| `dryRun` | используется для тестов, если стоит dryRun=true то софт не делает реальные сделки но выводит лог как будто правда работает, по умолчанию false |
| `telegramBotToken` | от [@BotFather](https://t.me/BotFather) |
| `telegramChatId` | твой айди (узнать у [@userinfobot](https://t.me/userinfobot)). |
| `telegramProxyUrl` | прокси для тг, чтобы бот мог отправлять сообщения с сервера с любым гео, если ничего не писать, то возьмёт первую проксю из файла `proxies.txt` |

#### Безопасность

- `pk.txt`, `config.json`, `proxies.txt`, `lighter-keys.json`,
  `rotation-state.json` — никому не отправляйте, в них приватные ключи и данные аккаунтов
