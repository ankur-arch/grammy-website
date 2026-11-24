---
prev: false
next: false
---

# Hosting: Prisma Compute

This guide tells you how you can host your grammY bots on [Prisma Compute](https://www.prisma.io/compute).

Prisma Compute runs TypeScript apps on [Bun](https://bun.sh), so this guide uses the `grammy` package from npm together with the `bun` [adapter](../guide/deployment-types#web-framework-adapters).
An app that receives no requests is put to sleep, and it wakes up again when the next request comes in.
This means that you need to [run your bot on webhooks](../guide/deployment-types#how-to-use-webhooks), and that long polling does not work.
It also means that a network connection which your bot keeps open, for example to a database, may no longer work after your bot wakes up, so your code should be able to open it again.

## Prerequisites

To follow along, you need to have a [Prisma account](https://console.prisma.io).
You also need to have [Node.js](https://nodejs.org) and [Bun](https://bun.sh) installed on your machine.
The Prisma CLI runs on Node.js 24.11 or newer (or 22.18 or newer if you are using Node.js 22), and Bun is used to bundle your bot.

## Setup

Initialize your project and install the dependencies.
In addition to grammY, you need [Prisma Composer](https://www.prisma.io/docs/composer), which is used to describe your app in TypeScript, and the Prisma CLI, which deploys it.

```sh
# Initialize the project.
mkdir grammy-bot
cd grammy-bot
npm init -y
npm pkg set type=module

# Install main dependencies.
npm install grammy @prisma/composer @prisma/composer-prisma-cloud arktype

# Install development dependencies.
npm install -D typescript @types/bun prisma
```

Add a `build` script to your `package.json`.
It bundles your bot into a single file, which is what gets deployed.

```json
{
  "scripts": {
    "build": "bun build src/bot.ts --target=bun --outfile dist/bot.mjs" // [!code focus]
  }
}
```

Create a `tsconfig.json` file with the following content.

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "Preserve",
    "moduleResolution": "bundler",
    "allowImportingTsExtensions": true,
    "noEmit": true,
    "strict": true,
    "skipLibCheck": true,
    "types": ["bun"]
  },
  "include": ["module.ts", "prisma.config.ts", "src"]
}
```

Once you have created all files from this guide, your project's root directory should look like this:

```asciiart:no-line-numbers
.
├── node_modules/
├── src/
│   ├── bot.ts
│   └── service.ts
├── module.ts
├── package.json
├── package-lock.json
├── prisma.config.ts
└── tsconfig.json
```

## Preparing Your Code

> Remember that you need to [run your bot on webhooks](../guide/deployment-types#how-to-use-webhooks), so you should use `webhookCallback` and not call `bot.start()` in your code.

Create a file named `src/bot.ts`.
You can use this short example bot as a starting point.

```ts
import { Bot, webhookCallback } from "grammy";
import service from "./service.ts";

const token = service.input().token.expose();

const bot = new Bot(token);

bot.command("start", (ctx) => ctx.reply("Welcome! Up and running."));
bot.on("message:text", (ctx) => ctx.reply(ctx.message.text));

const handleUpdate = webhookCallback(bot, "bun");

Bun.serve({
  port: service.port(),
  hostname: "0.0.0.0",
  async fetch(req) {
    const url = new URL(req.url);
    if (req.method === "POST" && url.pathname.slice(1) === bot.token) {
      try {
        return await handleUpdate(req);
      } catch (err) {
        console.error(err);
      }
    }
    return new Response();
  },
});
```

We advise you to have your handler on some secret path rather than the root (`/`).
Here, we are using the bot token (`/<bot token>`).

Note that the server has to listen on the port returned by `service.port()` and on the hostname `0.0.0.0`, because otherwise Prisma Compute cannot reach it.
The bot token is not read from an environment variable.
Instead, it is passed in by the service that you are going to declare in the next step.

## Describing Your App

Prisma Compute needs three small files that describe your app.

The first one is `src/service.ts`.
It names the service, points to the bundled file, and declares that the service needs a secret called `token`.

```ts
import { secretString } from "@prisma/composer/arktype";
import node from "@prisma/composer/node";
import { compute } from "@prisma/composer-prisma-cloud";
import { type } from "arktype";

export default compute({
  name: "bot",
  deps: {},
  input: type({ token: secretString() }),
  build: node({ module: import.meta.url, entry: "../dist/bot.mjs" }),
});
```

The second one is `module.ts` in the root directory of your project.
It is the app itself, and its name is also the name of your project on Prisma Compute.
It also says that the value of `token` is stored on Prisma Compute in a variable named `BOT_TOKEN`.

```ts
import { appRoot } from "@prisma/composer";
import { variable } from "@prisma/composer-prisma-cloud";
import botService from "./src/service.ts";

export default appRoot("grammy-bot", ({ provision }) => {
  const token = provision(variable.secret({ name: "BOT_TOKEN" }));
  provision(botService, { input: { token } });
});
```

The third one is `prisma.config.ts`, also in the root directory.
It configures the deployment.
The `region` is where your bot is going to run.

```ts
import { defineConfig as composer } from "@prisma/composer/config";
import { nodeBuild } from "@prisma/composer/node/control";
import {
  prismaCloud,
  prismaState,
} from "@prisma/composer-prisma-cloud/control";
import { definePrismaConfig } from "prisma/config";

export default definePrismaConfig({
  composer: composer({
    entry: "./module.ts",
    extensions: [prismaCloud({ region: "eu-central-1" }), nodeBuild()],
    state: prismaState(),
  }),
});
```

## Deploying

Sign in to your Prisma account.
This command opens your browser.

```sh
npx prisma auth login
```

Create a project on Prisma Compute, and give your bot token to it.
Use the same name as in `module.ts`, and the same region as in `prisma.config.ts`.

```sh
npx prisma project create grammy-bot --region eu-central-1

# Replace 123:aBcDeF-gh with your real bot token.
npx prisma project env add 'BOT_TOKEN=123:aBcDeF-gh' --role production
```

The Prisma CLI does not build your bot for you, so you need to build it first, and then deploy it.

```sh
npm run build
npx prisma deploy
```

When the deployment is done, the public URL of your bot is printed.

```log:no-line-numbers
grammy-bot.bot: https://<SERVICE_ID>.fra.prisma.build
✔ Deployed grammy-bot to production in 21.2s.
```

## Setting the Webhook URL

Your bot is now running.
All that's left to do is to tell Telegram where to send the updates.
You can do this by calling `setWebhook`.
For example, open a new tab in your browser and visit this URL:

```text
https://api.telegram.org/bot<BOT_TOKEN>/setWebhook?url=https://<SERVICE_ID>.fra.prisma.build/<BOT_TOKEN>
```

Replace both occurrences of `<BOT_TOKEN>` with your real bot token.
Replace `<SERVICE_ID>.fra.prisma.build` with the hostname that was printed by the `deploy` command.

You should now see this in your browser window.

```json
{ "ok": true, "result": true, "description": "Webhook was set" }
```

Done!
Your bot is working now.
Head over to Telegram and watch it respond to messages!

If your bot does not respond, you can read its logs by running `npx prisma service logs grammy-bot.bot`.
