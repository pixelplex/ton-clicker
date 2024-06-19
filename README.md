# Clicker frontend App

The project is based on the Next.js framework. You can find more information about it on the official [Next.js website](https://nextjs.org/).

## Development

Clone the repository and install dependencies by running the following command:

```bash
yarn install
```

After installing the dependencies, you can start the development server with:

```bash
yarn dev
```

This command runs the app in development mode.

Open [http://localhost:10888](http://localhost:10888) to view it in the browser.

The page will automatically reload if you make changes to the code.

### Additional Steps for Development

After starting the app in development mode, you need to run the following command to create a tunnel:

```bash
yarn tunnel
```

Use the generated link for VK and Telegram mini apps.

## Run in Production

### Build and Run in Production Mode Manually

You can build the bundle and run the application in production mode with the following commands:

```bash
yarn build:prod // creates bundle in the .next folder
yarn start      // starts the app in production mode on port 3000
```

# Migrating MiniApps from VK to Telegram

## 1. Getting Started

### Creating a Bot in Telegram

To start working with MiniApps in Telegram, you need to create a bot. Follow these steps:

1. Open the Telegram app and find BotFather.
2. Start a dialogue with BotFather and use the `/start` command to begin creating a bot.
3. Enter the `/newbot` command and follow the instructions to create a new bot. You will need to choose a name and a unique username for the bot.
4. After successfully creating the bot, BotFather will provide you with an access token, which will be required for interacting with the Telegram API.

### Registering and Setting Up a MiniApp

To register and set up a MiniApp, follow these steps:

1. Go to the [Creating a Telegram Web App](https://core.telegram.org/bots/webapps) section in the official Telegram documentation.
2. Follow the instructions to register your MiniApp, set up the necessary parameters, and integrate it with the bot.

## 2. Interaction with Platform APIs

Specialized libraries are used to interact with platform APIs, providing convenient access to the functions and features of these platforms. VKontakte and Telegram provide such libraries, namely `vk-bridge` and `@tma.js/sdk`, respectively. Both libraries perform similar functions, allowing developers to interact with their platform's APIs to obtain user data and perform other tasks.

### VKontakte: vk-bridge

The `vk-bridge` library is designed to interact with the VKontakte API. The official documentation can be found [here](https://dev.vk.com/ru/bridge/overview).


### Telegram: @tma.js/sdk

The `@tma.js/sdk` library is designed to interact with the Telegram API. The official documentation can be found [here](https://docs.telegram-mini-apps.com/packages/tma-js-sdk).

# App authorization

## 3. VK

Main difference between vk and telegram is how you authorize your user.

### Backend
Vk does not have any custom library to authorize. U need to manually calculate hash of `signParams` with secret key,
provided in vk miniapp settings.

```ts
const VK_APP_SECRET_KEY = 'VK_APP_SECRET_KEY';

function isSignValid(sign: string, signParams: Record<string, string>): boolean {
  const signUrlParams = new URLSearchParams(signParams);
  signUrlParams.sort();

  const queryString = signUrlParams.toString();

  const paramsHash = crypto
    .createHmac('sha256', VK_APP_SECRET_KEY)
    .update(queryString)
    .digest()
    .toString('base64url');

  return paramsHash === sign;
}
```

After validating sign params you can extract user data from sign params. For example `vkUserId`.

```ts 
const vkUserId = signParams.vk_user_id;
```

After this just place `vkUserId` in database or anywhere. 

### Frontend

Use `vk-bridge` to get sign and sign params data.

```ts
  const { sign, ...signParams } = await bridge.send('VKWebAppGetLaunchParams');
```

## 4. Telegram

### Backend

Telegram has similar mechanism. But instead of manual validation you can use package `@tma.js/init-data-node` to validate `initData`,
using secret key, provided `@BotFather`.

```ts
import { validate } from '@tma.js/init-data-node';

const TG_BOT_SECRET = 'TG_BOT_SECRET';

function isInitDataValid(initDataRaw: string): boolean {
  try {
    validate(initDataRaw, TG_BOT_SECRET);
    return true;
  } catch (err) {
    return false;
  }
}
```

After validating init data you can extract user information using `@tms.js/init-data-node`.

```ts
import { parse } from '@tma.js/init-data-node';

const initData = parse(initDataRaw);
const tgUserId = initData.user.id.toString();
```

After this just place `tgUserId` in database or anywhere.

### Frontend

Use `@tma.js/sdk` to get `initDataRaw`.

```ts
  import { retrieveLaunchParams } from '@tma.js/sdk';

  const { initDataRaw } = retrieveLaunchParams();
```

## 5. Auth with Ton wallet (optional)

Also, you can authorize your user using Ton Wallet.
Standard way is using Ton Proof. Below is the example. More information you can find on [official docs](https://docs.ton.org/develop/dapps/ton-connect/sign).

### Backend

```ts
export async function isProofValid(payload: TonProof): Promise<boolean> {
  try {
    const stateInit = loadStateInit(Cell.fromBase64(payload.proof.stateInit).beginParse());
    const publicKey = tryParsePublicKey(stateInit);
    if (!publicKey) {
      return false;
    }

    const walletPublicKey = Buffer.from(payload.publicKey, 'hex');
    if (!publicKey.equals(walletPublicKey)) {
      return false;
    }

    const address = Address.parse(payload.address);
    const walletAddress = contractAddress(address.workChain, stateInit);
    if (!walletAddress.equals(address)) {
      return false;
    }

    if (!ALLOWED_DOMAINS.includes(payload.proof.domain.value)) {
      return false;
    }

    const now = Math.floor(Date.now() / 1000);
    if (now - VALID_AUTH_TIME > payload.proof.timestamp) {
      return false;
    }

    const message = {
      workchain: walletAddress.workChain,
      address: walletAddress.hash,
      domain: {
        lengthBytes: payload.proof.domain.lengthBytes,
        value: payload.proof.domain.value,
      },
      signature: Buffer.from(payload.proof.signature, 'base64'),
      payload: payload.proof.payload,
      stateInit: payload.proof.stateInit,
      timestamp: payload.proof.timestamp,
    };

    const wc = Buffer.alloc(4);
    wc.writeUInt32BE(message.workchain, 0);

    const ts = Buffer.alloc(8);
    ts.writeBigUInt64LE(BigInt(message.timestamp), 0);

    const dl = Buffer.alloc(4);
    dl.writeUInt32LE(message.domain.lengthBytes, 0);

    const msg = Buffer.concat([
      Buffer.from(TON_PROOF_PREFIX),
      wc,
      message.address,
      dl,
      Buffer.from(message.domain.value),
      ts,
      Buffer.from(message.payload),
    ]);

    const msgHash = Buffer.from(await sha256(msg));

    const fullMsg = Buffer.concat([
      Buffer.from([0xff, 0xff]),
      Buffer.from(TON_CONNECT_PREFIX),
      msgHash,
    ]);

    const result = Buffer.from(await sha256(fullMsg));

    return sign.detached.verify(result, message.signature, publicKey);
  } catch (e) {
    return false;
  }
}
```

