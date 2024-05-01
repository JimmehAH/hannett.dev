---
layout: single
title:  "Bindicator Part 1"
date:   2024-04-19 15:00:00 +0000
categories: projects bindicator
excerpt: What is a bindicator, why do I want one and why do I have to make it myself?
toc: true
toc_sticky: true
---

## The Problem

I would like to have a glanceable reminder of when to put the bins out and which bins need to go out. Currently I have 3 options to check the bin day in my area;

1. Enter my address on the website that [Oxford City Council](https://www.oxford.gov.uk) provide and see what comes back. As you can see in the image below it's not the most readable layout and it's certainly not convenient or glanceable. Not sure why they bother to include the details of the next collection and also the collection after that.
  ![Screenshot of a website showing bin collection dates. The text is small, there is no particular order to the data](/assets/images/oxford_bin_collection.png)
2. Print out a PDF of all the collection dates for the year and print it out. This is also not very glanceable, although it would be slightly more convenient.
  ![Cropped screenshot of a PDF that lists all the bin collection days in my area for the year. There's a lot of info packed in](/assets/images/oxford_bin_zones.png)
3. Get an email reminder from the Council. This is the option I currently use and it's certainly the best one from a convenience and glanceability perspective. As you can see in the screenshot only the pertinent information is given and as the email arrives on my phone it's very convenient. The only issue is that the email tends to arrive during the mid-afternoon and I'm usually working or doing something at that time. By the time I get home I've often forgotten about the email and then when I **do** remember it's 10pm and I have to go digging through the emails again to remind myself which bins go out.
  ![Cropped screenshot of an email showing the next bin collection. It only shows the relevant information and is quite readable](/assets/images/oxford_bin_email.png)

Wouldn't it be better if I could just have some sort of small light that would come on to remind me? Nothing too garish, of course.

![Photo of a clear glass skull shaped bottled stuffed full of multicoloured LEDs](/assets/images/plasma_skull.jpeg)

This is a Pi Pico powered Skull which I bought from [Pimoroni](https://shop.pimoroni.com/products/wireless-plasma-kit?variant=40362173399123) because it's a WiFI enabled glass skull crammed full of RGB LEDs.

I think that if I have this on a shelf somewhere and have it light up in the evenings with the relevant bin colour (blue for recycling, green for refuse) then that will serve the purpose of providing a glanceable and convenient reminder of when to put the bins out.

So I have my skull with WiFi, how should I make it glow when I want it to? Making it glow is not too difficult, as you can see from these [code examples](https://github.com/pimoroni/pimoroni-pico/tree/main/micropython/examples/plasma_stick). Making it glow at the right time will require some thought.

I am not the first person to want a device like this. I am not even the first person to use the term '[bindicator](https://twitter.com/tarbard/status/1002464120447397888?lang=en)'. There are even some commercial products available such as '[The Bindicator](https://www.bindicator.net)', although that one seems to rely on using a phone app to set a schedule rather than pulling the data automatically.

Pulling the data automatically is the other problem we'll need to solve to make this truly convenient. I don't want to faff around with setting up schedules using a phone app, or maintaining some sort of online calendar because then I'll have to remember to update them when there's a bank holiday or extreme weather event and the council have to change the bin collection day.

Oxford used to have a mobile app that had an API that could be relatively easily reverse engineered and repurposed as you can see in [Terence Eden's Alexa Bin Day blog post](https://shkspr.mobi/blog/2017/07/alexa-what-bin-day-is-it/) but that has been shut down now. I suppose they stopped using Cloud9's platform to power their online portals.

There is at least 1 open source project devoted to providing a standard interface to the various disparate council bin collection data sources. It's called [UK Bin Collection Data (UKBCD)](https://github.com/robbrad/UKBinCollectionData) but sadly it does not support Oxford City Council yet.

![Screenshot of a list of supported councils for the UK Bin Collection Data integration for Home Assistant. Oxford is not on the list.](/assets/images/oxford_bin_homeassistant.png)

We're going to have to do this ourselves.

## The Solution

We will use the email reminder option that the council provide and have it send emails to a service such as [Zapier](https://zapier.com) or [IFTTT](https://ifttt.com). From there we can forward the body of the email to a custom web service which will process it and store the relevant details (date and type of next collection). We will also write some custom code for the skull which will query this web service a couple of times a day to get the current latest collection date and time, along with updating the internal clock of the Pico. Between 6pm and 11pm on the day before the collection the skull will glow a relevant colour. We will secure access to the web service using some sort of [pre-shared key](https://en.wikipedia.org/wiki/Pre-shared_key).

That's the idea, anyway. We'll see how I got on in part II.
