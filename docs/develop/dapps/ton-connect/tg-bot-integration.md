# Telegram bot integration

In this tutorial, we’ll create a sample telegram bot that supports TON Connect 2.0 authentication.
We will analyze connecting a wallet, sending a transaction, getting data about the connected wallet, and disconnecting a wallet.

## Documentation links

1. [TON Connect SDK documentation](https://www.npmjs.com/package/@tonconnect/sdk)  
2. [Integration example source code](https://github.com/ton-connect/demo-telegram-bot)

## Prerequisites

- You need to create a telegram bot using [@BotFather](https://t.me/BotFather) and save its token.
- Node JS should be installed (we use version 18.1.0 in this tutorial).
- Docker should be installed.


## Creating project

### Set up dependencies
Firstly we have to create a Node JS project. We will use TypeScript and [node-telegram-bot-api](https://www.npmjs.com/package/node-telegram-bot-api) library (you can choose any other suitable for you library as well). Also, we will use [qrcode](https://www.npmjs.com/package/qrcode) library for QR codes generation, but you can replace it with any other same library.

Let's create a directory `ton-connect-bot`. Add the following package.json file there:
```json
{
  "name": "ton-connect-bot",
  "version": "1.0.0",
  "scripts": {
    "compile": "npx rimraf dist && tsc",
    "run": "node ./dist/main.js"
  },
  "dependencies": {
    "@tonconnect/sdk": "^2.1.3",
    "dotenv": "^16.0.3",
    "node-telegram-bot-api": "^0.61.0",
    "qrcode": "^1.5.1"
  },
  "devDependencies": {
    "@types/node-telegram-bot-api": "^0.61.4",
    "@types/qrcode": "^1.5.0",
    "rimraf": "^3.0.2",
    "typescript": "^4.9.5"
  }
}
```

Run `npm i` to install dependencies.

### Add a tsconfig.json
Create a `tsconfig.json`:

<details>
<summary>tsconfig.json code</summary>

```json
{
  "compilerOptions": {
    "declaration": true,
    "lib": ["ESNext", "dom"],
    "resolveJsonModule": true,
    "experimentalDecorators": false,
    "esModuleInterop": true,
    "allowSyntheticDefaultImports": true,
    "target": "es6",
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,
    "noUnusedLocals": false,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "module": "commonjs",
    "moduleResolution": "node",
    "sourceMap": true,
    "useUnknownInCatchVariables": false,
    "noUncheckedIndexedAccess": true,
    "emitDecoratorMetadata": false,
    "importHelpers": false,
    "skipLibCheck": true,
    "skipDefaultLibCheck": true,
    "allowJs": true,
    "outDir": "./dist"
  },
  "include": ["src"],
  "exclude": [
    "./tests","node_modules", "lib", "types"]
}
```
</details>

[Read more about tsconfig.json](https://www.typescriptlang.org/docs/handbook/tsconfig-json.html)

### Add simple bot code
Create a `.env` file and add your bot token, dApp manifest and wallets list cache time to live there:

[See more about tonconnect-manifes.json](https://github.com/ton-connect/sdk/tree/main/packages/sdk#add-the-tonconnect-manifest)

```dotenv
# .env
TELEGRAM_BOT_TOKEN=<YOUR BOT TOKEN, LIKE 1234567890:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA>
MANIFEST_URL=https://raw.githubusercontent.com/ton-connect/demo-telegram-bot/master/tonconnect-manifest.json
WALLETS_LIST_CAHCE_TTL_MS=86400000
```

Create directory `src` and file `bot.ts` inside. Let's create a TelegramBot instance there:

```ts
// src/bot.ts

import TelegramBot from 'node-telegram-bot-api';
import * as process from 'process';

const token = process.env.TELEGRAM_BOT_TOKEN!;

export const bot = new TelegramBot(token, { polling: true });
```

Now we can create an entrypoint file `main.ts` inside the `src` directory:

```ts
// src/main.ts
import dotenv from 'dotenv';
dotenv.config();

import { bot } from './bot';

bot.on('message', msg => {
  const chatId = msg.chat.id;

  bot.sendMessage(chatId, 'Received your message');
});
```

Here we go. You can run `npm run compile` and `npm run start`, and send any message to your bot. Bot will reply "Received your message". We are ready for the TonConnect integration.

At the moment we have the following files structure:
```text
ton-connect-bot
├── src
│   ├── bot.ts
│   └── main.ts
├── package.json
├── package-lock.json
├── .env
└── tsconfig.json
```

## Connecting a wallet
We've already installed the `@tonconnect/sdk`, so we can just import it to get started.

We will start with getting wallets list. We need only http-bridge-compatible wallets. Create a folder `ton-connect` into `src` and add `wallets.ts` file there:

```ts
// src/ton-connect/wallets.ts

import { isWalletInfoRemote, WalletInfoRemote, WalletsListManager } from '@tonconnect/sdk';

const walletsListManager = new WalletsListManager({
    cacheTTLMs: Number(process.env.WALLETS_LIST_CAHCE_TTL_MS)
});

export async function getWallets(): Promise<WalletInfoRemote[]> {
    const wallets = await walletsListManager.getWallets();
    return wallets.filter(isWalletInfoRemote);
}
```

Now we need to define a TonConnect storage. TonConnect uses `localStorage` to save connection details when running in the browser, however there is no `localStorage` in NodeJS environment. That's why we should add a custom simple storage implementation.

[See details about TonConnect storage](https://github.com/ton-connect/sdk/tree/main/packages/sdk#init-connector)

Create `storage.ts` inside `ton-connect` directory:

```ts
// src/ton-connect/storage.ts

import { IStorage } from '@tonconnect/sdk';

const storage = new Map<string, string>(); // temporary storage implementation. We will replace it with the redis later

export class TonConnectStorage implements IStorage {
  constructor(private readonly chatId: number) {} // we need to have different stores for different users 

  private getKey(key: string): string {
    return this.chatId.toString() + key; // we will simply have different keys prefixes for different users
  }

  async removeItem(key: string): Promise<void> {
    storage.delete(this.getKey(key));
  }

  async setItem(key: string, value: string): Promise<void> {
    storage.set(this.getKey(key), value);
  }

  async getItem(key: string): Promise<string | null> {
    return storage.get(this.getKey(key)) || null;
  }
}
```

We are moving on implementing a wallet connection.
Modify `src/main.ts` and add `connect` command. We are going to implement a wallet connection in this command handler.
```ts
import dotenv from 'dotenv';
dotenv.config();

import { bot } from './bot';
import { getWallets } from './ton-connect/wallets';
import TonConnect from '@tonconnect/sdk';
import { TonConnectStorage } from './ton-connect/storage';
import QRCode from 'qrcode';

bot.onText(/\/connect/, async msg => {
  const chatId = msg.chat.id;
  const wallets = await getWallets();

  const connector = new TonConnect({
    storage: new TonConnectStorage(chatId),
    manifestUrl: process.env.MANIFEST_URL
  });

  connector.onStatusChange(wallet => {
    if (wallet) {
      bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
    }
  });

  const tonkeeper = wallets.find(wallet => wallet.name === 'Tonkeeper')!;

  const link = connector.connect({
    bridgeUrl: tonkeeper.bridgeUrl,
    universalLink: tonkeeper.universalLink
  });
  const image = await QRCode.toBuffer(link);

  await bot.sendPhoto(chatId, image);
});
```

Let's analyze what we are doing here. Firstly we fetch the wallets list and create a TonConnect instance.
After that we subscribe to wallet change. When user connects a wallet, bot will send a message `${wallet.device.appName} wallet connected!`.
Next we find the Tonkeeper wallet and create connection link. In the end we generate a QR code with the link and send it as a photo to the user.

Now you can run the bot (`npm run compile` and `npm run start` then) and send `/connect` message to the bot. Bot should reply with the QR. Scan it with the Tonkeeper wallet. You will see a message `Tonkeeper wallet connected!` in the chat. 


We will use connector in many places, so let's move connector creating code to a separate file:
```ts
// src/ton-connect/conenctor.ts

import TonConnect from '@tonconnect/sdk';
import { TonConnectStorage } from './storage';
import * as process from 'process';

export function getConnector(chatId: number): TonConnect {
    return new TonConnect({
        manifestUrl: process.env.MANIFEST_URL,
        storage: new TonConnectStorage(chatId)
    });
}
```

And import it in the `src/main.ts`

```ts
// src/main.ts

import dotenv from 'dotenv';
dotenv.config();

import { bot } from './bot';
import { getWallets } from './ton-connect/wallets';
import QRCode from 'qrcode';
import { getConnector } from './ton-connect/connector';

bot.onText(/\/connect/, async msg => {
    const chatId = msg.chat.id;
    const wallets = await getWallets();

    const connector = getConnector(chatId);

    connector.onStatusChange(wallet => {
        if (wallet) {
            bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
        }
    });

    const tonkeeper = wallets.find(wallet => wallet.name === 'Tonkeeper')!;

    const link = connector.connect({
        bridgeUrl: tonkeeper.bridgeUrl,
        universalLink: tonkeeper.universalLink
    });
    const image = await QRCode.toBuffer(link);

    await bot.sendPhoto(chatId, image);
});
```

At the moment we have the following files structure:
```text
bot-demo
├── src
│   ├── ton-connect
│   │   ├── connector.ts
│   │   ├── wallets.ts
│   │   └── storage.ts
│   ├── bot.ts
│   └── main.ts
├── package.json
├── package-lock.json
├── .env
└── tsconfig.json
```

## Creating connect wallet menu
### Add inline keyboard
We've done the Tonkeeper wallet connection. But we didn't implement connection via universal QR code for all wallets, and didn't allow the user to choose suitable wallet. Let's cover it now.

For better UX we are going to use `callback_query` and `inline_keyboard` Telegram features. If you don't fill familiar with that, you can read more about it [here](https://core.telegram.org/bots/api#callbackquery).

We will implement following UX for wallet connection:
```text
First screen:
<Unified QR>
<Open wallet unified link>, <Choose a wallet button (opens second screen)> 

Second screen:
<Unified QR>
<Back (opens first screen)>
<Tonkeeper button (opens third screen)>, <Tonhub button (opens third screen)>, <...> 

Third screen:
<Selected wallet QR>
<Back (opens second screen)>
<Open selected wallet link> 
```

Let's start with adding inline keyboard to the `/connect` command handler in the `main.ts`
```ts
// src/main.ts
bot.onText(/\/connect/, async msg => {
    const chatId = msg.chat.id;
    const wallets = await getWallets();

    const connector = getConnector(chatId);

    connector.onStatusChange(wallet => {
        if (wallet) {
            bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
        }
    });

    const link = connector.connect(wallets);
    const image = await QRCode.toBuffer(link);

    await bot.sendPhoto(chatId, image, {
        reply_markup: {
            inline_keyboard: [
                [
                    {
                        text: 'Open Wallet',
                        url: `https://ton-connect.github.io/open-tc?connect=${encodeURIComponent(
                            link
                        )}`
                    },
                    {
                        text: 'Choose a Wallet',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    }
                ]
            ]
        }
    });
});
```

We need to wrap TonConnect deeplink as https://ton-connect.github.io/open-tc?connect=${encodeURIComponent(link)} because only `http` links are allowed in the Telegram inline keyboard.
Website https://ton-connect.github.io/open-tc just redirects user to link passed in the `connect` query param, so it's only workaround to open `tc://` link in the Telegram.

Note that we replaced `connector.connect` call arguments. Now we are generating a unified link for all wallets. 

Next we tell Telegram to call `callback_query` handler with `{ "method": "chose_wallet" }` value when user clicks to the `Choose a Wallet` button.

### Add Choose a Wallet button handler
Create a file `src/connect-wallet-menu.ts`.

Let's add 'Choose a Wallet' button click handler there:

```ts
// src/connect-wallet-menu.ts

async function onChooseWalletClick(query: CallbackQuery, _: string): Promise<void> {
    const wallets = await getWallets();

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                wallets.map(wallet => ({
                    text: wallet.name,
                    callback_data: JSON.stringify({ method: 'select_wallet', data: wallet.name })
                })),
                [
                    {
                        text: '« Back',
                        callback_data: JSON.stringify({
                            method: 'universal_qr'
                        })
                    }
                ]
            ]
        },
        {
            message_id: query.message!.message_id,
            chat_id: query.message!.chat.id
        }
    );
}
```
Here we are replacing the message inline keyboard with a new one that contains clickable list of wallets and 'Back' button.


Now we will add global `callback_query` handler and register `onChooseWalletClick` there:
```ts
// src/connect-wallet-menu.ts
import { CallbackQuery } from 'node-telegram-bot-api';
import { getWallets } from './ton-connect/wallets';
import { bot } from './bot';

export const walletMenuCallbacks = { // Define buttons callbacks
    chose_wallet: onChooseWalletClick
};

bot.on('callback_query', query => { // Parse callback data and execute corresponing function 
    if (!query.data) {
        return;
    }

    let request: { method: string; data: string };

    try {
        request = JSON.parse(query.data);
    } catch {
        return;
    }

    if (!walletMenuCallbacks[request.method as keyof typeof walletMenuCallbacks]) {
        return;
    }

    walletMenuCallbacks[request.method as keyof typeof walletMenuCallbacks](query, request.data);
});

// ... other code from the previous ster
async function onChooseWalletClick ...
```

Here we define buttons handlers list and `callback_query` parser. Unfortunately callback data is always string, so we have to pass JSON to the `callback_data` and parse it later in the `callback_query` handler. 
Then we are looking for the requested method and call it with passed parameters.

Now we should add `conenct-wallet-menu.ts` import to the `main.ts`
```ts
// src/main.ts

// ... other imports 

import './connect-wallet-menu';

// ... other code
```

Compile and run the bot. You can click to the Choose a wallet button and bot will replace inline keyboard buttons!

### Add other buttons handlers
Let's complete this menu and add rest commands handlers.

Firstly we will create a utility function `editQR`. Editing message media (QR image) is a bit tricky. We need to store image to the file and send it to the Telegram server. Then we can remove this file.


```ts
// src/connect-wallet-menu.ts

// ... other code


async function editQR(message: TelegramBot.Message, link: string): Promise<void> {
    const fileName = 'QR-code-' + Math.round(Math.random() * 10000000000);

    await QRCode.toFile(`./${fileName}`, link);

    await bot.editMessageMedia(
        {
            type: 'photo',
            media: `attach://${fileName}`
        },
        {
            message_id: message?.message_id,
            chat_id: message?.chat.id
        }
    );

    await new Promise(r => fs.rm(`./${fileName}`, r));
}
```


In `onOpenUniversalQRClick` handler we just regenerate a QR and deeplink and modify the message:
```ts
// src/connect-wallet-menu.ts

// ... other code

async function onOpenUniversalQRClick(query: CallbackQuery, _: string): Promise<void> {
    const chatId = query.message!.chat.id;
    const wallets = await getWallets();

    const connector = getConnector(chatId);

    connector.onStatusChange(wallet => {
        if (wallet) {
            bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
        }
    });

    const link = connector.connect(wallets);

    await editQR(query.message!, link);

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                [
                    {
                        text: 'Open Wallet',
                        url: `https://ton-connect.github.io/open-tc?connect=${encodeURIComponent(
                            link
                        )}`
                    },
                    {
                        text: 'Choose a Wallet',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    }
                ]
            ]
        },
        {
            message_id: query.message?.message_id,
            chat_id: query.message?.chat.id
        }
    );
}

// ... other code
```


In `onWalletClick` handler we are creating special QR and universal link for selected wallet only, and modify the message.
```ts
// src/connect-wallet-menu.ts

// ... other code

async function onWalletClick(query: CallbackQuery, data: string): Promise<void> {
    const chatId = query.message!.chat.id;
    const connector = getConnector(chatId);

    connector.onStatusChange(wallet => {
        if (wallet) {
            bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
        }
    });

    const wallets = await getWallets();

    const selectedWallet = wallets.find(wallet => wallet.name === data);
    if (!selectedWallet) {
        return;
    }

    const link = connector.connect({
        bridgeUrl: selectedWallet.bridgeUrl,
        universalLink: selectedWallet.universalLink
    });

    await editQR(query.message!, link);

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                [
                    {
                        text: '« Back',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    },
                    {
                        text: `Open ${data}`,
                        url: link
                    }
                ]
            ]
        },
        {
            message_id: query.message?.message_id,
            chat_id: chatId
        }
    );
}

// ... other code
```

Now we have to register this functions as callbacks (`walletMenuCallbacks`):

```ts
// src/connect-wallet-menu.ts
import TelegramBot, { CallbackQuery } from 'node-telegram-bot-api';
import { getWallets } from './ton-connect/wallets';
import { bot } from './bot';
import * as fs from 'fs';
import { getConnector } from './ton-connect/connector';
import QRCode from 'qrcode';

export const walletMenuCallbacks = {
    chose_wallet: onChooseWalletClick,
    select_wallet: onWalletClick,
    universal_qr: onOpenUniversalQRClick
};

// ... other code
```
<details>
<summary>Currently src/connect-wallet-menu.ts looks like that</summary>

```ts
// src/connect-wallet-menu.ts

import TelegramBot, { CallbackQuery } from 'node-telegram-bot-api';
import { getWallets } from './ton-connect/wallets';
import { bot } from './bot';
import { getConnector } from './ton-connect/connector';
import QRCode from 'qrcode';
import * as fs from 'fs';

export const walletMenuCallbacks = {
    chose_wallet: onChooseWalletClick,
    select_wallet: onWalletClick,
    universal_qr: onOpenUniversalQRClick
};

bot.on('callback_query', query => { // Parse callback data and execute corresponing function 
    if (!query.data) {
        return;
    }

    let request: { method: string; data: string };

    try {
        request = JSON.parse(query.data);
    } catch {
        return;
    }

    if (!callbacks[request.method as keyof typeof callbacks]) {
        return;
    }

    callbacks[request.method as keyof typeof callbacks](query, request.data);
});


async function onChooseWalletClick(query: CallbackQuery, _: string): Promise<void> {
    const wallets = await getWallets();

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                wallets.map(wallet => ({
                    text: wallet.name,
                    callback_data: JSON.stringify({ method: 'select_wallet', data: wallet.name })
                })),
                [
                    {
                        text: '« Back',
                        callback_data: JSON.stringify({
                            method: 'universal_qr'
                        })
                    }
                ]
            ]
        },
        {
            message_id: query.message!.message_id,
            chat_id: query.message!.chat.id
        }
    );
}

async function onOpenUniversalQRClick(query: CallbackQuery, _: string): Promise<void> {
    const chatId = query.message!.chat.id;
    const wallets = await getWallets();

    const connector = getConnector(chatId);

    connector.onStatusChange(wallet => {
        if (wallet) {
            bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
        }
    });

    const link = connector.connect(wallets);

    await editQR(query.message!, link);

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                [
                    {
                        text: 'Open Wallet',
                        url: `https://ton-connect.github.io/open-tc?connect=${encodeURIComponent(
                            link
                        )}`
                    },
                    {
                        text: 'Choose a Wallet',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    }
                ]
            ]
        },
        {
            message_id: query.message?.message_id,
            chat_id: query.message?.chat.id
        }
    );
}

async function onWalletClick(query: CallbackQuery, data: string): Promise<void> {
    const chatId = query.message!.chat.id;
    const connector = getConnector(chatId);

    connector.onStatusChange(wallet => {
        if (wallet) {
            bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
        }
    });

    const wallets = await getWallets();

    const selectedWallet = wallets.find(wallet => wallet.name === data);
    if (!selectedWallet) {
        return;
    }

    const link = connector.connect({
        bridgeUrl: selectedWallet.bridgeUrl,
        universalLink: selectedWallet.universalLink
    });

    await editQR(query.message!, link);

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                [
                    {
                        text: '« Back',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    },
                    {
                        text: `Open ${data}`,
                        url: link
                    }
                ]
            ]
        },
        {
            message_id: query.message?.message_id,
            chat_id: chatId
        }
    );
}

async function editQR(message: TelegramBot.Message, link: string): Promise<void> {
    const fileName = 'QR-code-' + Math.round(Math.random() * 10000000000);

    await QRCode.toFile(`./${fileName}`, link);

    await bot.editMessageMedia(
        {
            type: 'photo',
            media: `attach://${fileName}`
        },
        {
            message_id: message?.message_id,
            chat_id: message?.chat.id
        }
    );

    await new Promise(r => fs.rm(`./${fileName}`, r));
}
```
</details>

Compile and run the bot to check how wallet connection works now.

You may note that we haven't considered QR code expiration and stopping connectors yet. We will handle it later. 


At the moment we have the following files structure:
```text
bot-demo
├── src
│   ├── ton-connect
│   │   ├── connector.ts
│   │   ├── wallets.ts
│   │   └── storage.ts
│   ├── bot.ts
│   ├── connect-wallet-menu.ts
│   └── main.ts
├── package.json
├── package-lock.json
├── .env
└── tsconfig.json
```

## Implement transaction sending
Before write new code that sends a transaction, let's clean up the code. We're going to create a new file for bot commands handlers ('/connect', '/send_tx', ...)

```ts
// src/commands-handlers.ts

import { bot } from './bot';
import { getWallets } from './ton-connect/wallets';
import QRCode from 'qrcode';
import TelegramBot from 'node-telegram-bot-api';
import { getConnector } from './ton-connect/connector';

export async function handleConnectCommand(msg: TelegramBot.Message): Promise<void> {
    const chatId = msg.chat.id;
    const wallets = await getWallets();

    const connector = getConnector(chatId);

    connector.onStatusChange(wallet => {
        if (wallet) {
            bot.sendMessage(chatId, `${wallet.device.appName} wallet connected!`);
        }
    });

    const link = connector.connect(wallets);
    const image = await QRCode.toBuffer(link);

    await bot.sendPhoto(chatId, image, {
        reply_markup: {
            inline_keyboard: [
                [
                    {
                        text: 'Open Wallet',
                        url: `https://ton-connect.github.io/open-tc?connect=${encodeURIComponent(
                            link
                        )}`
                    },
                    {
                        text: 'Choose a Wallet',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    }
                ]
            ]
        }
    });
}
```

Let's import that in the `main.ts` and also move `callback_query` entrypoint from `connect-wallet-menu.ts` to the `main.ts`:

```ts
// src/main.ts

import dotenv from 'dotenv';
dotenv.config();

import { bot } from './bot';
import './connect-wallet-menu';
import { handleConnectCommand } from './commands-handlers';
import { walletMenuCallbacks } from './connect-wallet-menu';

const callbacks = {
    ...walletMenuCallbacks
};

bot.on('callback_query', query => {
    if (!query.data) {
        return;
    }

    let request: { method: string; data: string };

    try {
        request = JSON.parse(query.data);
    } catch {
        return;
    }

    if (!callbacks[request.method as keyof typeof callbacks]) {
        return;
    }

    callbacks[request.method as keyof typeof callbacks](query, request.data);
});

bot.onText(/\/connect/, handleConnectCommand);
```

```ts
// src/connect-wallet-menu.ts

// ... imports


export const walletMenuCallbacks = {
    chose_wallet: onChooseWalletClick,
    select_wallet: onWalletClick,
    universal_qr: onOpenUniversalQRClick
};

async function onChooseWalletClick(query: CallbackQuery, _: string): Promise<void> {
    
// ... other code
```

Now we can add `send_tx` command handler:

```ts
// src/commands-handlers.ts

// ... other code

export async function handleSendTXCommand(msg: TelegramBot.Message): Promise<void> {
    const chatId = msg.chat.id;

    const connector = getConnector(chatId);

    await connector.restoreConnection();
    if (!connector.connected) {
        await bot.sendMessage(chatId, 'Connect wallet to send transaction');
        return;
    }

    const wallets = await connector.getWallets();

    connector
        .sendTransaction({
            validUntil: Math.round((Date.now() + 600000) / 1000), // timeout is SECONDS
            messages: [
                {
                    amount: '1000000',
                    address: '0:0000000000000000000000000000000000000000000000000000000000000000'
                }
            ]
        })
        .then(() => {
            bot.sendMessage(chatId, `Transaction sent successfully`);
        })
        .catch(e => {
            if (e instanceof UserRejectsError) {
                bot.sendMessage(chatId, `You rejected the transaction`);
                return;
            }

            bot.sendMessage(chatId, `Unknown error happened`);
        })
        .finally(() => connector.pauseConnection());

    let deeplink = '';
    const walletInfo = wallets.find(wallet => wallet.name === connector.wallet!.device.appName);
    if (walletInfo && isWalletInfoRemote(walletInfo)) {
        deeplink = walletInfo.universalLink;
    }

    await bot.sendMessage(
        chatId,
        `Open ${connector.wallet!.device.appName} and confirm transaction`,
        {
            reply_markup: {
                inline_keyboard: [
                    [
                        {
                            text: 'Open Wallet',
                            url: deeplink
                        }
                    ]
                ]
            }
        }
    );
}
```

Here we check if user's wallet is connected and process sending transaction.
Then we send a message to the user with a button that opens user's wallet (wallet universal link without additional parameters).
Note that this button contains an empty deeplink. It means that send transaction request data goes through the http-bridge, and transaction will appear it the user's wallet even if he just open the wallet app without clicking to this button.

Let's register this handler:
```ts
// src/main.ts

// ... other code

bot.onText(/\/connect/, handleConnectCommand);

bot.onText(/\/send_tx/, handleSendTXCommand);
```

Compile and run the bot to check that transaction sending works correctly.

At the moment we have the following files structure:
```text
bot-demo
├── src
│   ├── ton-connect
│   │   ├── connector.ts
│   │   ├── wallets.ts
│   │   └── storage.ts
│   ├── bot.ts
│   ├── connect-wallet-menu.ts
│   ├── commands-handlers.ts
│   └── main.ts
├── package.json
├── package-lock.json
├── .env
└── tsconfig.json
```

## Add disconnect and show connected wallet commands
This commands implementation is simple enough:

```ts
// src/commands-handlers.ts

// ... other code

export async function handleDisconnectCommand(msg: TelegramBot.Message): Promise<void> {
    const chatId = msg.chat.id;

    const connector = getConnector(chatId);

    await connector.restoreConnection();
    if (!connector.connected) {
        await bot.sendMessage(chatId, "You didn't connect a wallet");
        return;
    }

    await connector.disconnect();

    await bot.sendMessage(chatId, 'Wallet has been disconnected');
}

export async function handleShowMyWalletCommand(msg: TelegramBot.Message): Promise<void> {
    const chatId = msg.chat.id;

    const connector = getConnector(chatId);

    await connector.restoreConnection();
    if (!connector.connected) {
        await bot.sendMessage(chatId, "You didn't connect a wallet");
        return;
    }

    await bot.sendMessage(
        chatId,
        `Connected wallet: ${
            connector.wallet!.device.appName
        }\nYour address: ${toUserFriendlyAddress(
            connector.wallet!.account.address,
            connector.wallet!.account.chain === CHAIN.TESTNET
        )}`
    );
}
```

And register this commands:
```ts
// src/main.ts

// ... other code

bot.onText(/\/connect/, handleConnectCommand);
bot.onText(/\/send_tx/, handleSendTXCommand);
bot.onText(/\/disconnect/, handleDisconnectCommand);
bot.onText(/\/my_wallet/, handleShowMyWalletCommand);
```

Compile and run the bot to check that commands above works correctly.

## Optimisation
We've done all basic commands. But it is important to keep in mind that each connector keeps SSE connection opened until it is paused. 
Also, we didn't handle case when user calls `/connect` multiple times, or calls `/connect` or `/send_tx` and doesn't scan the QR. We should set a timeout and close the connection to save server resources. 
Then we should notify user that QR / transaction request is expired.

### Send transaction optimisation
Let's create a utility function that wraps a promise and rejects it after the specified timeout:

```ts
// src/utils.ts

export const pTimeoutException = Symbol();

export function pTimeout<T>(
    promise: Promise<T>,
    time: number,
    exception: unknown = pTimeoutException
): Promise<T> {
    let timer: ReturnType<typeof setTimeout>;
    return Promise.race([
        promise,
        new Promise((_r, rej) => (timer = setTimeout(rej, time, exception)))
    ]).finally(() => clearTimeout(timer)) as Promise<T>;
}
```

You can use this code or pick a library you like.

Let's add a timeout parameter value to the `.env`

```dotenv
# .env
TELEGRAM_BOT_TOKEN=<YOUR BOT TOKEN, LIKE 1234567890:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA>
MANIFEST_URL=https://raw.githubusercontent.com/ton-connect/demo-telegram-bot/master/tonconnect-manifest.json
WALLETS_LIST_CAHCE_TTL_MS=86400000
DELETE_SEND_TX_MESSAGE_TIMEOUT_MS=600000
```

Now we are going to improve `handleSendTXCommand` function and wrap tx sending to the `pTimeout`

```ts
// src/commands-handlers.ts

// export async function handleSendTXCommand(msg: TelegramBot.Message): Promise<void> { ...

pTimeout(
    connector.sendTransaction({
        validUntil: Math.round(
            (Date.now() + Number(process.env.DELETE_SEND_TX_MESSAGE_TIMEOUT_MS)) / 1000
        ),
        messages: [
            {
                amount: '1000000',
                address: '0:0000000000000000000000000000000000000000000000000000000000000000'
            }
        ]
    }),
    Number(process.env.DELETE_SEND_TX_MESSAGE_TIMEOUT_MS)
)
    .then(() => {
        bot.sendMessage(chatId, `Transaction sent successfully`);
    })
    .catch(e => {
        if (e === pTimeoutException) {
            bot.sendMessage(chatId, `Transaction was not confirmed`);
            return;
        }

        if (e instanceof UserRejectsError) {
            bot.sendMessage(chatId, `You rejected the transaction`);
            return;
        }

        bot.sendMessage(chatId, `Unknown error happened`);
    })
    .finally(() => connector.pauseConnection());

// ... other code
```

<details>
<summary>Full handleSendTXCommand code</summary>

```ts
export async function handleSendTXCommand(msg: TelegramBot.Message): Promise<void> {
    const chatId = msg.chat.id;

    const connector = getConnector(chatId);

    await connector.restoreConnection();
    if (!connector.connected) {
        await bot.sendMessage(chatId, 'Connect wallet to send transaction');
        return;
    }

    const wallets = await connector.getWallets();

    pTimeout(
        connector.sendTransaction({
            validUntil: Math.round(
                (Date.now() + Number(process.env.DELETE_SEND_TX_MESSAGE_TIMEOUT_MS)) / 1000
            ),
            messages: [
                {
                    amount: '1000000',
                    address: '0:0000000000000000000000000000000000000000000000000000000000000000'
                }
            ]
        }),
        Number(process.env.DELETE_SEND_TX_MESSAGE_TIMEOUT_MS)
    )
        .then(() => {
            bot.sendMessage(chatId, `Transaction sent successfully`);
        })
        .catch(e => {
            if (e === pTimeoutException) {
                bot.sendMessage(chatId, `Transaction was not confirmed`);
                return;
            }

            if (e instanceof UserRejectsError) {
                bot.sendMessage(chatId, `You rejected the transaction`);
                return;
            }

            bot.sendMessage(chatId, `Unknown error happened`);
        })
        .finally(() => connector.pauseConnection());

    let deeplink = '';
    const walletInfo = wallets.find(wallet => wallet.name === connector.wallet!.device.appName);
    if (walletInfo && isWalletInfoRemote(walletInfo)) {
        deeplink = walletInfo.universalLink;
    }

    await bot.sendMessage(
        chatId,
        `Open ${connector.wallet!.device.appName} and confirm transaction`,
        {
            reply_markup: {
                inline_keyboard: [
                    [
                        {
                            text: 'Open Wallet',
                            url: deeplink
                        }
                    ]
                ]
            }
        }
    );
}
```

</details>

If user doesn't confirm the transaction during `DELETE_SEND_TX_MESSAGE_TIMEOUT_MS` (10min), the transaction will be cancelled and bot will send a message `Transaction was not confirmed`.

You can set this parameter to `5000` compile and rerun the bot and test its behaviour.

### Wallet connect flow optimisation
At this moment we create a new connector on the every navigation through the wallet connection menu step.
That is poorly because we don't close previous connectors connection when create new connectors.
Let's improve this behaviour and create a cache-mapping for users connectors.

<details>
<summary>src/ton-connect/connector.ts code</summary>

```ts
// src/ton-connect/connector.ts

import TonConnect from '@tonconnect/sdk';
import { TonConnectStorage } from './storage';
import * as process from 'process';

type StoredConnectorData = {
    connector: TonConnect;
    timeout: ReturnType<typeof setTimeout>;
    onConnectorExpired: ((connector: TonConnect) => void)[];
};

const connectors = new Map<number, StoredConnectorData>();

export function getConnector(
    chatId: number,
    onConnectorExpired?: (connector: TonConnect) => void
): TonConnect {
    let storedItem: StoredConnectorData;
    if (connectors.has(chatId)) {
        storedItem = connectors.get(chatId)!;
        clearTimeout(storedItem.timeout);
    } else {
        storedItem = {
            connector: new TonConnect({
                manifestUrl: process.env.MANIFEST_URL,
                storage: new TonConnectStorage(chatId)
            }),
            onConnectorExpired: []
        } as unknown as StoredConnectorData;
    }

    if (onConnectorExpired) {
        storedItem.onConnectorExpired.push(onConnectorExpired);
    }

    storedItem.timeout = setTimeout(() => {
        if (connectors.has(chatId)) {
            const storedItem = connectors.get(chatId)!;
            storedItem.connector.pauseConnection();
            storedItem.onConnectorExpired.forEach(callback => callback(storedItem.connector));
            connectors.delete(chatId);
        }
    }, Number(process.env.CONNECTOR_TTL_MS));

    connectors.set(chatId, storedItem);
    return storedItem.connector;
}
```

</details>

This code may look a little tricky, but here we go. 
Here we store a connector, it's cleaning timeout and list of callback that should be executed after the timeout for each user.

When `getConnector` is called we check if there is an existing connector for this `chatId` (user) it the cache. If it exists we reset the cleaning timeout and return the connector. 
That allows keep active users connectors in cache. It there is no connector in the cache we create a new one, register a timeout clean function and return this connector.

To make it works we have to add a new parameter to the `.env`

```dotenv
# .env
TELEGRAM_BOT_TOKEN=<YOUR BOT TOKEN, LIKE 1234567890:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA>
MANIFEST_URL=https://ton-connect.github.io/demo-dapp-with-react-ui/tonconnect-manifest.json
WALLETS_LIST_CAHCE_TTL_MS=86400000
DELETE_SEND_TX_MESSAGE_TIMEOUT_MS=600000
CONNECTOR_TTL_MS=600000
```

Now let's use it in the handelConnectCommand

<details>
<summary>src/commands-handlers.ts code</summary>

```ts
// src/commands-handlers.ts

import {
    CHAIN,
    isWalletInfoRemote,
    toUserFriendlyAddress,
    UserRejectsError
} from '@tonconnect/sdk';
import { bot } from './bot';
import { getWallets } from './ton-connect/wallets';
import QRCode from 'qrcode';
import TelegramBot from 'node-telegram-bot-api';
import { getConnector } from './ton-connect/connector';
import { pTimeout, pTimeoutException } from './utils';

let newConnectRequestListenersMap = new Map<number, () => void>();

export async function handleConnectCommand(msg: TelegramBot.Message): Promise<void> {
    const chatId = msg.chat.id;
    let messageWasDeleted = false;

    newConnectRequestListenersMap.get(chatId)?.();

    const connector = getConnector(chatId, () => {
        unsubscribe();
        newConnectRequestListenersMap.delete(chatId);
        deleteMessage();
    });

    await connector.restoreConnection();
    if (connector.connected) {
        await bot.sendMessage(
            chatId,
            `You have already connected a ${
                connector.wallet!.device.appName
            } wallet\nYour address: ${toUserFriendlyAddress(
                connector.wallet!.account.address,
                connector.wallet!.account.chain === CHAIN.TESTNET
            )}\n\n Disconnect wallet firstly to connect a new one`
        );

        return;
    }

    const unsubscribe = connector.onStatusChange(async wallet => {
        if (wallet) {
            await deleteMessage();

            await bot.sendMessage(chatId, `${wallet.device.appName} wallet connected successfully`);
            unsubscribe();
            newConnectRequestListenersMap.delete(chatId);
        }
    });

    const wallets = await getWallets();

    const link = connector.connect(wallets);
    const image = await QRCode.toBuffer(link);

    const botMessage = await bot.sendPhoto(chatId, image, {
        reply_markup: {
            inline_keyboard: [
                [
                    {
                        text: 'Open Wallet',
                        url: `https://ton-connect.github.io/open-tc?connect=${encodeURIComponent(
                            link
                        )}`
                    },
                    {
                        text: 'Choose a Wallet',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    }
                ]
            ]
        }
    });

    const deleteMessage = async (): Promise<void> => {
        if (!messageWasDeleted) {
            messageWasDeleted = true;
            await bot.deleteMessage(chatId, botMessage.message_id);
        }
    };

    newConnectRequestListenersMap.set(chatId, async () => {
        unsubscribe();

        await deleteMessage();

        newConnectRequestListenersMap.delete(chatId);
    });
}

// ... other code
```

</details>

We defined `newConnectRequestListenersMap` to store cleanup callback for the last connect request for each user. 
If user calls `/connect` multiple times, bot will delete previous message with QR. 
Also, we subscribed to the connector expiration timeout to delete the QR-code message when it is expired.  


Now we should remove `connector.onStatusChange` subscription from the `connect-wallet-menu.ts` functions,
because they use the same connector instance and one subscription in the `handleConnectCommand` in enough.

<details>
<summary>src/connect-wallet-menu.ts code</summary>

```ts
// src/connect-wallet-menu.ts

// ... other code

async function onOpenUniversalQRClick(query: CallbackQuery, _: string): Promise<void> {
    const chatId = query.message!.chat.id;
    const wallets = await getWallets();

    const connector = getConnector(chatId);

    const link = connector.connect(wallets);

    await editQR(query.message!, link);

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                [
                    {
                        text: 'Open Wallet',
                        url: `https://ton-connect.github.io/open-tc?connect=${encodeURIComponent(
                            link
                        )}`
                    },
                    {
                        text: 'Choose a Wallet',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    }
                ]
            ]
        },
        {
            message_id: query.message?.message_id,
            chat_id: query.message?.chat.id
        }
    );
}

async function onWalletClick(query: CallbackQuery, data: string): Promise<void> {
    const chatId = query.message!.chat.id;
    const connector = getConnector(chatId);

    const wallets = await getWallets();

    const selectedWallet = wallets.find(wallet => wallet.name === data);
    if (!selectedWallet) {
        return;
    }

    const link = connector.connect({
        bridgeUrl: selectedWallet.bridgeUrl,
        universalLink: selectedWallet.universalLink
    });

    await editQR(query.message!, link);

    await bot.editMessageReplyMarkup(
        {
            inline_keyboard: [
                [
                    {
                        text: '« Back',
                        callback_data: JSON.stringify({ method: 'chose_wallet' })
                    },
                    {
                        text: `Open ${data}`,
                        url: link
                    }
                ]
            ]
        },
        {
            message_id: query.message?.message_id,
            chat_id: chatId
        }
    );
}

// ... other code
```

</details>

That's it! Compile and run the bot and try to call `/connect` twice.

## Add a permanent storage
At this moment we store TonConnect sessions in the Map object. But you may want to store it to the database or other permanent storage to save the sessions when you restart the server.
We will use Redis for that, but you can pick any permanent storage.

### Set up redis
Firstly run `npm i redis`.

[See package details](https://www.npmjs.com/package/redis)

To work with redis you have to start redis server. We will use the Docker image:
`docker run -p 6379:6379 -it redis/redis-stack-server:latest`

Now add redis connection parameter to the `.env`. Default redis url is `redis://127.0.0.1:6379`.
```dotenv
# .env
TELEGRAM_BOT_TOKEN=<YOUR BOT TOKEN, LIKE 1234567890:AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA>
MANIFEST_URL=https://ton-connect.github.io/demo-dapp-with-react-ui/tonconnect-manifest.json
WALLETS_LIST_CAHCE_TTL_MS=86400000
DELETE_SEND_TX_MESSAGE_TIMEOUT_MS=600000
CONNECTOR_TTL_MS=600000
REDIS_URL=redis://127.0.0.1:6379
```

Let's integrate redis to the `TonConnectStorage`:

```ts
// src/ton-connect/storage.ts

import { IStorage } from '@tonconnect/sdk';
import { createClient } from 'redis';

const client = createClient({ url: process.env.REDIS_URL });

client.on('error', err => console.log('Redis Client Error', err));

export async function initRedisClient(): Promise<void> {
    await client.connect();
}
export class TonConnectStorage implements IStorage {
    constructor(private readonly chatId: number) {}

    private getKey(key: string): string {
        return this.chatId.toString() + key;
    }

    async removeItem(key: string): Promise<void> {
        await client.del(this.getKey(key));
    }

    async setItem(key: string, value: string): Promise<void> {
        await client.set(this.getKey(key), value);
    }

    async getItem(key: string): Promise<string | null> {
        return (await client.get(this.getKey(key))) || null;
    }
}
```

To make it work we have to wait for the redis initialisation in the `main.ts`. Let's wrap code in this file to an async function:
```ts
// src/main.ts
// ... imports

async function main(): Promise<void> {
    await initRedisClient();

    const callbacks = {
        ...walletMenuCallbacks
    };

    bot.on('callback_query', query => {
        if (!query.data) {
            return;
        }

        let request: { method: string; data: string };

        try {
            request = JSON.parse(query.data);
        } catch {
            return;
        }

        if (!callbacks[request.method as keyof typeof callbacks]) {
            return;
        }

        callbacks[request.method as keyof typeof callbacks](query, request.data);
    });

    bot.onText(/\/connect/, handleConnectCommand);

    bot.onText(/\/send_tx/, handleSendTXCommand);

    bot.onText(/\/disconnect/, handleDisconnectCommand);

    bot.onText(/\/my_wallet/, handleShowMyWalletCommand);
}

main();
```

## Summary

What is next?
- If you want to run the bot in production you may want to install and use a process manager like [pm2](https://pm2.keymetrics.io/).
- You can add better errors handling in the bot.

## See Also
- [Sending messages](/develop/dapps/ton-connect/transactions)
- [Integration Manual](/develop/dapps/ton-connect/integration)
