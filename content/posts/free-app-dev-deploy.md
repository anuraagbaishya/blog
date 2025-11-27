---
title: "Using Free Resources to Build and Deploy Apps"
date: 2025-11-25T10:48:19-06:00
draft: false
toc: true
images:
tags:
  - development
---

Recently, an app building bug bit me. This was a result of me finding an app that solved a problem I was facing, but it was not free. For a long time, the apps I built for myself were CLI only, as I did not have the skills or the patience to develop a web UI (especially writing HTML and CSS). A CLI would however not work for the usecase in question.

Due to the not so recent advent of AI, I had the idea of running a few experiments. If you are one of the two people who read my blog, you would know that I am miserly - and therefore try to make do with free resources as much as I can. These experiments were no different. I used the free tier of every resource I used. The app development experiment is broadly divided into two parts - developing and deploying. I used basic [ChatGPT web](https://chatgpt.com/) for code assistance, [Gemini APIs](https://ai.google.dev/) for "AI functionalities" and [Vercel](https://vercel.com) + [MongoDB Atlas](https://www.mongodb.com/products/platform/atlas-database) for free deployments.

I know there are better tools to do this. IDEs such as Cursor also have free plans. However, I was unsure if I would run out of the free tier limits - and therefore, decided to use ChatGPT, which does not have any limits. I did run out of GPT 5 queries very quickly - but overall that was not a factor in my experimentation, as the small number of GPT 5 queries I made, compared to GPT 4, would not make it a fair assessment.

## Development
The app I set on building is a recipe app. I generally cook my own meals and therefore have a bunch of recipes bookmarked / screenshotted / in my notes app(s). I decide what I want to cook for the week ahead on the weekends and then get groceries accordingly. My existing setup has an annoying shortcoming. I have to manually go through the recipes I want to cook and create my shopping list. To somewhat solve this problem, I have a very long note on my notes app (that has balloned to ~50-60 items) where I mark the items I need to buy. This works okay, but as the list keeps getting longer and longer, finding items I need to mark has become tedious. 

I tried finding an app which could solve this problem - and there are quite a few. But all of them require me to use some premium feature to achieve the desired results. As you may have guessed, I did not want to pay. So, I decided to build my own app. 

### Experiment 1.1 - the backend
The first experiment was to use ChatGPT to build the backend. Since I have built a fair amount of API servers, the plan was to only use ChatGPT to simplify certain functions, add typing, and fix the odd error that I was not able to fix quickly. Basically things I know how to do - but I am lazy to. This also meant I could provide precise instructions to ChatGPT to help me out.

I settled on using FastAPI to build the APIs. I have decent experience working with FastAPI - having used it to build production grade APIs at Adobe. Using ChatGPT as needed - I was able to quickly build the APIs. ChatGPT (as expected) did great with the precise instructions to add typing or fix an error.

Given enough different segments of code in different chat messages, ChatGPT was able to eventually piece together how the API architecture looked and what the endpoints were - and was able to provide accurate suggestions. These suggestions for the most part worked directly on copy-paste. Sometimes it was over-helpful and provided suggestions to improve other parts of the code which I did not ask for. I used these extra suggestions as I saw fit. 

Overral experience: 10/10.

### Experiment 1.2 - the frontend
At first, I decided to build the web UI using vanilla HTML / CSS / JS. HTML / CSS is not my strong suit, but I have used them to varying degrees. JS is now probably my second most used language after Python - particularly from using it to build production grade servers at Palo Alto Networks. However, frontend specific JS such as dynamically creating HTML or adding CSS to elements is not something I have a lot of experience with.

My first challenge was to explain how I wanted the UI to look to ChatGPT in words. This did not prove to be as big a challenge as I was expecting. ChatGPT understood quite well what I was trying to build. ChatGPT used a handy trick in one of its responses which I then adapted to explain UI though text. Suppose I want a list of items, each of which have a text element and two buttons. The trick is to use a block like
```
[name of recipe          |edit button|delete button]
```
to represent how I want the elements placed. ChatGPT was able to understand this and provide code to make the list look more or less as I wanted it to.

The more challenging part was to fix minor spacing and alignment issues. ChatGPT seemed to go around in circles when asked to fix these issues (which could not be represented well in the text representation). Eventually after a bit of back and forth - and very verbose explanations of what is going wrong - it did provide good enough suggestions. Other times I would just ask for generic CSS properties, such as "how to center a button in a div", to sort-of break the circular answers, and try to apply the properties myself. Using a combination of all of the above, I was able to build a decent looking UI.

At some point I decided I have had enough of adding weird dynamic HTML / CSS using JS to achieve the desired UI and pivotted to using React. I have a basic understanding of how React works - but not enough to build an entire UI with multiple pages / components / routing / etc. I was not starting from scratch though. I provided the existing HTML and JS I had, and asked ChatGPT to convert it to React components as it saw fit. Once again, ChatGPT did a great job, breaking down the HTML / JS to reusable components, re-organizing the logic, and providing correct navigation properties for the app to work correctly. I was able to reuse the CSS I already had for the most part. The few changes needed were made manually or again with assistance from ChatGPT.

Overall experience - 7.5/10 (other than the CSS parts I would say it was 9/10. It's just not as 
straighforward to explain desired UIs in words.)

### Experiment 1.3 - the iOS app
Going from the backend — which I could write completely by myself if I wasn’t lazy — to the UI, which I could do about 50% of, the next logical step is to build something I’m completely unfamiliar with: an iOS app. I have, in a past life, built many Android apps but that was about 8 years ago. Therefore, I don't want to claim I have experience with developing apps.

Why an iOS app? Well, the web UX was good enough to use on a laptop - but I would not be going around with a laptop while grocery shopping. The UX on mobile, on the other hand, was terrible. While I could have used ChatGPT to make the web UI responsive, the premise of getting to play with new shiny things such as XCode and Swift excited me more than wrangling with CSS / JS again. 

I used the React components as the baseline and asked ChatGPT to convert them to Swift UI. Along the way I asked ChatGPT some extremely noob questions on iOS app development. To no surprise, ChatGPT again handled this task without any hiccups. Once the basic skeleton was done, I got into the more iOS specific functionalities - liquid glass effects, pull to refresh, navigation from one view to another, splash screens, the whole nine yards. 

The challenge here was two fold:
1. I did not know how to ask for things. I did not know iOS specific terminologies, and had to resort to using generic words and describing actions, hoping ChatGPT would understand. For the most part it did, very well.
2. I could not fix even the smallest bugs myself as I did not know anything about the language or the framework. So I had to use ChatGPT for everything. Eventually I did get to a point where I could copy-paste code from other parts of the app and get that to work.

There were a couple instances where ChatGPT failed me. One particular issue was managing asynchronous API calls. I had ChatGPT implement pull to refresh for the lists but due to how it architected the asynchoronous calls, the `refreshable` action would not get a completion notification from the API call, and therefore would continue to spin endlessly. ChatGPT kept going around a circle and suggested the same solutions over and over again. After struggling for 30 minutes or so, I gave up and asked this very specific question to Gemini - which answered it in one go.

Experience: 9/10. Thanks ChatGPT for building the iOS app for me and also teaching me basic Swift. Onto adding iOS app developer to my resume...

### Experiment 1.4 - free AI APIs
I learnt from a friend that Gemini APIs had a [free tier](https://ai.google.dev/gemini-api/docs/pricing). Naturally, I had to somehow incorporate this into my app. I had three usecases:
1. Convert the measured ingredients to just the ingredients (eg: converting "1 tbsp black pepper" to "black pepper") so that they could be added to the shopping list.
2. Deriving the cuisine of the dish to use in a filter.
3. Generating recipes.

I used Gemini 2.5 Flash Lite to get a structured JSON response - which it handled very well.

Experience - 10/10

## Deploying
Once I had the iOS app up and running, I needed to deploy the APIs such that they were reachable outside my home network. I initially wanted to use my Raspberry Pi to host the app and then use something like a [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/) to expose it to the internet. After an initial excitement at using yet more shiny new things, the idea seemed a bit too insecure for my security engineer brain. In the past I have used Heroku to deploy apps for free, but sadly, Heroku removed their free tier (TBH, I did not check recently if it was reinstated). I have also used GCP and AWS sporadically but they are not free. Cheap enough (~$0.1 a month) for my wallet to not notice but not free. In my quest to find a free hosting platform, I landed on [Vercel](https://vercel.com).

### Experiment 2.1 - deploying to Vercel
Deploying to Vercel was mostly a breeze. I just had to import the Github repo to Vercel, add environment variables and secrets, and let Vercel do its thing. The deployment was complete within a few seconds. I did have to modify some parts of the API - especially the MongoDB connection part - but these were very minor changes.

Experience: 10/10

### Experiment 2.2 - free DB
The database technology I am most familiar with is MongoDB - which works brillianly since I am always dealing with JSON data. I used local MongoDB for all the local development and testing. MongoDB provides a fully managed cloud service [Atlas](https://www.mongodb.com/products/platform/atlas-database), that has a free tier. Atlas also [integrates with Vercel](https://vercel.com/marketplace/mongodbatlas) very easily. Once I set up a cluster on Atlas, I was able to seamlessly migrate my collections to Atlas. With this last step complete, my app + APIs could now fulfill all the requirements I had.

Experience: 10/10

## Impressions
In total I probably spent about 40 hours to develop the APIs, the webapp (which I later discarded) and the iOS app. Granted that the app is not production ready by any means, it still serves all my needs. Without ChatGPT, building the iOS app itself would have taken me 100+ hours. I am highly delighted with the outcome - and my brain is already thinking of more apps I can develop. One in particular that I am building is a web UI wrapper over Semgrep that fetches Github security advisories (GHSAs) and then lets me scan the repos associated with the GHSAs - allowing me to attempt finding other vulnerabilities in said repos.

I have heard a lot of people say that their reliance on using AI assistance to write code has negatively impacted their coding skills. This can be true. At a lot of points, I was tempted to ask ChatGPT to write code that I could write myself too - out of pure laziness. I would be lying if I said I did not give in to the temptation. Having said that, I made it a point to understand the code ChatGPT provided, and learn any new inbuilt functions or syntactic sugar it used. I also tried using these newly learnt patterns myself when I felt they were needed. This, in turn, improved my coding skills. For example, I can now use promises, array filters / maps in JS, and write basic React elements myself. 

## Conclusion
This will probably be the last blog of the year. Therefore, this conclusion section is more of a reflection on the year. I set out to find 5 CVEs this year. I started out strong with 3 reports within the first 3 months. Got 2 CVEs, 1 GHSA. I also decided to change the goal to 5 security issues as I learnt not everything I find may get a CVE. Somewhere along the line life happened and I did not work on security research much after the initial months. Subsequently, I will not be meeting my goal of 5 CVEs. I did, however, experiment a whole bunch with AI, both at work and in my personal time. Supplementing SAST with AI is my latest "passion". I have not done enough to write about it yet though. I also built apps, which was not in my to-do list for the year. 

(5 CVEs - 2 CVEs + 2 apps + 5 blogs + "AI experiments") =  success.

Thanks for reading. Happy holidays! See you all next year. I will not set goals for 2026 and instead "go with the flow". Let's see what the year has in store!