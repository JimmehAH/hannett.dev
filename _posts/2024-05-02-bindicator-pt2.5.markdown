---
layout: single
title:  "Bindicator Part 2.5"
date:   2024-05-08 16:00:00 +0000
categories: projects bindicator
excerpt: Cleaning up & enforcing software quality
toc: true
toc_sticky: true
---

## Intro

Hello! We still have some tidying up to do after the last part and maybe also make some small tweaks, before we move on the indicator itself.

## Tweaks

Doing some experimentation with the MicroPython environment for the skull has caused me to realise that it has no builtins for parsing an [RFC3339 timestamp](https://datatracker.ietf.org/doc/html/rfc3339). To save effort  we'll just add a type to represent the [8-tuple](https://docs.micropython.org/en/latest/library/time.html#time.mktime) that MicroPython prefers when creating DateTime objects. Unfortunately we can't use just some epoch time because the epoch date can vary depending on the flavour of MicroPython we are using.

```typescript
// see https://docs.micropython.org/en/latest/library/time.html#time.mktime for details of this 8-tuple
type MicropythonTimestamp = {
  year: number; // year includes the century (for example 2014).
  month: number; // month is 1-12
  mday: number; // mday is 1-31
  hour: number; // hour is 0-23
  minute: number; // minute is 0-59
  second: number; // second is 0-59
  weekday: number; // weekday is 0-6 for Mon-Sun
  yearday: number; // yearday is 1-366
};
```

Update the type to add a new optional field to our `WasteCollection` type called `mp_date`.

```typescript
type WasteCollection = {
  date: Date;
  mp_date?: MicropythonTimestamp;
  collections: CollectionType[];
};
```

I also changed the KV saving path to use an array of values, ordered from least-frequently changing to most-frequently. This matches [the style](https://docs.deno.com/deploy/kv/manual#creating-updating-and-reading-a-key-value-pair) used in the Deno docs.

```typescript
  await kv.set(
    ["next_collection", env["BASIC_USER"]],
    collection,
  );
```

I have also decided that some of the MicroPython code will be easier if I have a start date and end date. The Skull will only light up between these times.

Let's fix our `find_date_in_email` function and have it actually do something interesting. We know that the email will contain a day of the week.

![Screenshot of the email from the council showing the next collection day. The word 'FRIDAY' is highlighted.](/assets/images/oxford_bin_email_day.png)

We'll use a RegEx to pull out the day of the week we want. Interesting notes about the RegEx are that we are using the [multiline flag](https://javascript.info/regexp-multiline-mode) because what we're looking for is over multiple lines and we're also using a [named group](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Regular_expressions/Named_capturing_group) to make pulling out the day name later easier.

>Some people, when confronted with a problem, think
>“I know, I'll use regular expressions.”
>
>Now they have two problems.[^regex]

Then we'll iterate over today and the next 6 days. If we find a date that has a matching day name then we'll return that. If we don't find anything we'll default to returning tomorrow's date.

```typescript
function find_date_in_email(email_body: string): Date {
  // we only care about the date so set minutes and seconds to 0
  // we'll also set the hour to 6am now
  const now = dayjs().set("hour", 6).set("minute", 0).set("second", 0);

  const day_regex = RegExp(
    /Your next collection is on\s+(?<day_name>MONDAY|TUESDAY|WEDNESDAY|THURSDAY|FRIDAY|SATURDAY|SUNDAY)/,
    "gm",
  );

  const search = day_regex.exec(email_body)?.groups;

  // iterate over today and the next 7 days to find the correct date offset so we can make a proper timestamp for the collection
  // this will almost always be tomorrow
  if (search) {
    for (let date_offset = 0; date_offset < 7; date_offset++) {
      // add the offset to create our new date to test
      const calculated_date = now.add(date_offset, "day");
      console.log(calculated_date.format("dddd").toUpperCase());
      if (search.day_name == calculated_date.format("dddd").toUpperCase()) {
        return calculated_date.toDate();
      }
    }
  }

  // if the search was not successful then just set the date to tomorrow
  return now.add(1, "day").toDate();
}
```

We'll also do a little manipulation of the date when we create our `WasteCollection` object because if somehow it's a Friday and the collection is on the following Monday we only want the Bindicator to go off at a reasonable time on the Sunday.

```typescript
  const end_date = find_date_in_email(body.body);
  const start_date = dayjs(end_date).subtract(1, "day").set("hour", 16)
    .toDate();
```

While we're here we'll also refactor the web service to put all the authenticated routes behind a `/auth/*` prefix.

```typescript
app.use("/auth/*", basic_auth_mw);

app.post("/auth/incoming", async (c) => { ...snip... });
app.get("/auth/next-collection", async (c) => { ...snip... });
```

## Unit Testing

Let's add a couple of basic unit tests while we're here as well.

Open `deno.json` and add a `test` task.

```json
  "tasks": {
    "start": "deno run --allow-net --allow-read --watch --unstable-kv main.ts",
    "test": "deno test tests/"
  },
```

Create a new file called `main_test.ts` in a new `tests/` directory and put the following basic test in it.

```typescript
import { assertStringIncludes } from "https://deno.land/std@0.224.0/assert/mod.ts";
import app from "../main.ts";

Deno.test("basic test", async () => {
  const res = await app.request("/");

  assertStringIncludes(await res.text(), "Bindicator");
});
```

All this test does is check that the response from the default route contains the word 'Bindicator'.

Because we don't have mocking set up you'll need to use the following command to run the test: `deno test --allow-read --unstable-kv --allow-net`.

```shell
❯ deno test --allow-read --unstable-kv --allow-net
------- pre-test output -------
Listening on http://localhost:8000/
----- pre-test output end -----
running 1 test from ./tests/main_test.ts
basic test ... ok (5ms)

ok | 1 passed | 0 failed (9ms)
```

## CI/CD

Now we have a unit test we should set up some CI. We'll be using the [Continuous Integration](https://docs.deno.com/runtime/manual/advanced/continuous_integration) docs page from Deno as the basis of our work.

Create a new directory for our workflows.

```shell
mkdir -p .github/workflows
```

Add a file in there called `web-service.yml` and add the following.

```yaml
name: Check Web Service

on: push

jobs:
  build:
    runs-on: ubuntu-latest
    defaults:
        run:
          working-directory: ./web-service
    steps:
      - uses: actions/checkout@v3
      - uses: denoland/setup-deno@v1
        with:
          deno-version: v1.x # Run with latest stable Deno.
      # enforce codestyle
      - run: deno fmt --check
      # run lint check
      - run: deno lint
      # run tests
      - run: deno test --allow-read --unstable-kv --allow-net
```

The `working-directory` directive is necessary because we have 2 projects using different stacks in the same repo.

## Repo Setup

I would like to add some branch protection rules at this point. I will require pull requests and approvals before merges can be made to the `main` branch. I also turned on 'Require status checks to pass before merging'.

![Screenshot of GitHub showing that the 'Require PR' and 'Require approvals' branch protection settings have been turned on for the 'main' branch](/assets/images/oxford_bin_branch.png)

[^regex]: http://regex.info/blog/2006-09-15/247
