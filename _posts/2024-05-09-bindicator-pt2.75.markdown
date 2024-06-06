---
layout: single
title:  "Bindicator Part 2.75"
date:   2024-06-06 16:00:00 +0000
categories: projects bindicator
excerpt: Fixing the email -> webhook pipeline
toc: true
toc_sticky: true
---

## Intro

Turns out that webhooks on Zapier are __not__ free and in fact cost a minimum of $20 a month. That is way too much to pay for a single webhook that fires at most once per week so let's find an alternative.

## Cloudflare Email Workers

I'm already using Cloudflare for my domain registration and hosting for this site so let's try using their [email worker service](https://developers.cloudflare.com/email-routing/email-workers/).

As we're already using Deno I would have liked to have tried using Denoflare to write this but as of this time it doesn't seem to support the types necessary for writing an email worker. Maybe I'll try and fix that and rewrite it later.

In the meantime let's install [Node 20 LTS](https://nodejs.org/en/download/package-manager), then install [Yarn via npm](https://classic.yarnpkg.com/lang/en/docs/install/#mac-stable).

Because the `bindicator/` project root is not a Node project itself just run `npx wrangler generate email-router worker-typescript` to create a new project based on the [CloudFlare TypeScript worker sample](https://github.com/cloudflare/workers-sdk/tree/main/templates/worker-typescript).

Run `yarn` to install your dependencies. Run `yarn test` to make sure everything is working ok[^vitest].

At this point I also added `eslint` and `prettier` to the project, but that's up to you.

Add the `postal-mime` NPM package using `yarn add postal-mime` and add in a debug email handler to your `index.ts`.

```typescript
  async email(
    message: ForwardableEmailMessage,
    _env: Env,
    _ctx: ExecutionContext,
  ): Promise<void> {
    const emailParser = new PostalMime();

    const email = await emailParser.parse(message.raw);

    console.log(email.text);
  },
```

Now that we have something that can do something with an incoming email let's try it out for real. Log in to your [CloudFlare dashboard](https://dash.cloudflare.com/) and create a new worker. I called mine `bindicator-email-handler`. Go to Settings -> Triggers and [bind the worker to an email route](https://developers.cloudflare.com/email-routing/email-workers/enable-email-workers/).

Go back to your project and open `wrangler.toml`, set the name to the one you gave to your worker project.

Run `yarn deploy` and then `yarn run wrangler tail`. If you send an email to the address you bound to your worker you should see the body in the output from your `wrangler tail` session. Now we can start doing something actually useful with these incoming emails.

## Storing Incoming Emails

Create a new KV store using the CloudFlare Dash and make a note of the `id` and add this snippet to your `wranger.toml` file.

```toml
[[kv_namespaces]]
binding = "POST_DB"
id = "<your KV store id>"
```

Then open `src/index.ts` and change your `Env` declaration to match this:

```typescript
export interface Env {
  POST_DB: KVNamespace;
}
```

We're going to use the [`justatemp`](https://github.com/berrysauce/justatemp/tree/main) package as the basis for our email handling code as it's a good simple example of using both email workers and KV.

```typescript
    const email = await emailParser.parse(message.raw);

    const sender = email.from.address ?? "unknown";
    const recipient = email.to && email.to[0].address || "unknown";

    // use a random suffix to prevent future emails from clobbering
    const suffix = Math.random().toString(16).slice(2, 10);
    const key = recipient + "_" + suffix;

    const data = {
      suffix: suffix,
      recipient: recipient,
      sender: sender,
      subject: email.subject,
      "content-plain": email.text,
      date: email.date,
    };

    await env.POST_DB.put(key, JSON.stringify(data));
```

As you can see we have created a key for our key-value store based off the recipient and a short random string. This is to avoid having new emails that come in from overwriting the one already in the store. We can use a list operation and provide a prefix of just the recipient email address later to get all the emails. We'll want to keep the emails around for verification and maybe supporting multiple users later.

## Sending the email body to the Bindicator web service

I've pre-seeded the KV store with an object stored in `auth_<email address>`. That object has `auth_type` and `auth_token` fields. For now I've just reused the Basic auth with the credential we've already been using.

At this point you will try to use `fetch` and/or `axios` to make a POST request to the Bindicator service with the body of the email and spend __HOURS__ trying to work out why you keep getting this error: `TypeError: Too many redirects`. Eventually you find [this Stack Overflow post](https://stackoverflow.com/questions/68925550/cloudflare-workers-fetch-https-works-on-workers-dev-subdomain-but-not-on-own-sub/68946119#68946119) and do a little muffled scream in to a pillow.

> (Note: The "SSL Handshake fail" error itself is actually a known bug in Workers, that happens when you try to talk to a host outside your zone using HTTPS but you have "flexbile" SSL set. We do not use flexible SSL when talking to domains other than your own, but there's a bug that causes the request to fail instead of just using full SSL as it should.)

This has been an issue for over 2 years! Why does flipping a switch in my Cloudflare SSL configuration page fix this? The Bindicator service isn't even proxied via Cloudflare!

Anyway, flip the setting from 'Flexible' to 'Full' and get on with the rest of your life.

Before we send this off to our web service we should do an amount of due dilligance and ensure that we're not forwarding spam or something irrelevant.

```typescript
// make sure this email is the correct type (i.e. one about when the bins are going out)
if (email.subject?.includes("Your bins")) {
```

Grab our authentication data and build an Authorization header string. This should still work if we refactor to Bearer tokens.

```typescript
// get auth for this recipient from database
const auth_data: Auth_Data = JSON.parse(
  `${await env.POST_DB.get(`auth_${recipient}`)}`,
);
const auth_string = `${auth_data.auth_type} ${auth_data.auth_token}`;
```

Go ahead and POST the email body to the web service now. Also bear in mind that I'm telling a story and using code to illustrate it here. You can't just copy and paste this and expect it to work. Please check [the repo](https://github.com/JimmehAH/bindicator/) for an example of working code.

```typescript
const req_headers = new Headers();
req_headers.append("Content-Type", "application/json");
req_headers.append("Authorization", auth_string);

// post body of the email to the bindicator web service
const body = JSON.stringify({ body: email.text });

const req_options = {
  method: "POST",
  headers: req_headers,
  body: body,
  redirect: "follow",
};

const res = await fetch(env.BINDICATOR_ENDPOINT, req_options);
```

It would be a good idea to add something to clean up older emails after this point, which we might come back to in a future part.

Don't forget to update your email forwarding rule so that the incoming bin emails get sent to your new email handler. Later on we might add functionality to allow people to subscribe using a randomly generated address. Again, something for a future part.

## CI/CD

Let's use the GitHub Actions [wrangler action](https://github.com/marketplace/actions/deploy-to-cloudflare-workers-with-wrangler) to build and deploy our little email worker.

Create a [new Cloudflare API key](https://developers.cloudflare.com/workers/wrangler/ci-cd/). Use the `Edit Cloudflare Workers` template.

Create a [new GitHub Actions repository secret](https://docs.github.com/en/actions/security-guides/using-secrets-in-github-actions#creating-secrets-for-a-repository) called `CLOUDFLARE_API_KEY` and use the key you just got as the value. Create another called `CLOUDFLARE_ACCOUNT_ID` and use the account ID you can find on your Cloudflare dashboard in the Workers & Pages overview.

Create a new file called `email-router.yml` in `<project root>/.github/` (same directory that should have `web-service.yml` in it) and add this

```yaml
name: Deploy

on:
  push:
    paths:
    # only run this CI if something relevant has changed
      - 'email-router/*'

jobs:
  deploy:
    runs-on: ubuntu-latest
    defaults:
        run:
          working-directory: ./email-router
    name: Deploy
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
            node-version: 20
            cache: 'yarn'
      - run: yarn
      - run: yarn lint
      - name: Deploy
        # only deploy on changes to main branch
        if: github.ref == 'refs/heads/main'
        uses: cloudflare/wrangler-action@v3
        with:
          apiToken: ${{ secrets.CLOUDFLARE_API_KEY }}
          accountId: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
          workingDirectory: email-router/
          packageManager: yarn
```

[^vitest]: I had to run `yarn add -D vitest` here because the person who made this example probably had it installed globally and forgot to add it to their `package.json`.
