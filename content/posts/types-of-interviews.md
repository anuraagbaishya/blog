---
title: "Tales from my Product Security Job Hunt: Part 1, Interview Types"
date: 2024-01-28T16:12:46-08:00
toc: false
images:
tags:
  - interviews
---
After spending 4 amazing years at Adobe, I recently decided it was time to take the next step in my career. I was fortunate to interview with several companies that varied in terms of their domain, size, or the maturity of their security processes.

Many of those whom I informed about my decision and/or my interviews were interested in knowing what the process was like and how I prepared. After answering the same questions a few times, I thought it would be helpful to potential job seekers if I documented my experiences. Therefore, I am writing this two-part series that will talk about the types of interviews I attempted, and how I prepared for them. This blog in particular will list the types of interviews. In the next one, I will talk about my preparation and provide some tips that worked for me.

This list is specifically catered to the type of positions I was interviewing for. The interview rounds you have will vary according to your search criteria. My criteria:

* Experience: Mid-level (~4–5 years of experience)
* Role: Product Security / Application Security

## Interview Process
The interview process varied by company, but the general framework was:

* Recruiter call
* Basic technical interview
* Virtual panel-style interviews

In some cases, I spoke to the hiring manager before the first technical interview.

After the basic technical round, the next few rounds were often set up in a “virtual panel” style. This means they were set up concurrently (as opposed to a sequential setup where the next interview is set only if I perform well in the first).

Since the interview process varies a lot by company. I would recommend speaking with the recruiter regarding the process, specifically how many rounds to expect and what type of interviews they will be.

## Recruiter call
Typically, the first person you’d speak to will be a recruiter. In most cases, this is not an interview. The recruiter provides more information about the job role and the team. They might have a conversation about compensation expectations, visa requirements, and other administrative items. You may ask them for interview preparation guidance too. I have received great pointers regarding areas to concentrate on!

At times the recruiter also asked very basic security questions. Some questions include:

* What is (XSS, SQLi, CSRF, …, or any other common security issue)?
* Difference between encryption and hashing.
* How to store passwords in a DB.

## Basic Technical Interview
The first interview round was always a basic technical interview. We discussed common security issues and their mitigations (think OWASP top 10). Since I was interviewing for a mid-level position, I was expected to answer in-depth questions about security issues/concepts rather than just the definitions or basic mitigation steps. Some examples of questions I was asked:

* What are the types of XSS? How can you mitigate DOM XSS?
* How can you prevent CSRF in a case where we cannot use CSRF tokens?
* What is the principle behind using prepared statements for SQL injection mitigation?
* If using AES for encryption, which mode would you prefer between ECB, CBC, or GCM, and why?
* Explain a TLS handshake. (This seems to be a favorite amongst interviewers. I think this was the most commonly asked question.)
* What is the same origin policy and how does CORS relate to it? (Another favorite.)

The basic technical interview occasionally included questions about my resume (the other place where I was asked about my resume was during the manager interview). One of my interviewers and I shared a deep interest in static analysis and supply chain security — so we spent quite a bit of the interview talking about it!

## Manager Interview
I spoke with the hiring manager at the beginning of the application process for most of the roles. However, for a few positions, the manager interview occurred later, specifically during the panel interviews.

Generally, the conversation was aimed at understanding my general background and experience, and I was provided insights into what tasks I would work on if I joined. The general idea was to make sure my expectations were in line with what the role had to offer and vice-versa. Some included basic technical questions. Others included questions that tested the ability to think of scalable solutions or design initiatives. For eg: I was given some of the challenges the team faced and asked how I would approach them.

## Deeper Technical Interview
Wherever I made it to the next round, I had at least one deeper technical interview. This either consisted of going very deep into one or two common security issues or questions about security issues that are not as common (or not in the OWASP top 10). There might be scenario-based questions or domain-specific questions in this interview too. For eg, if you are applying to a role that requires knowledge of mobile application security, you will likely be asked about mobile-specific items.

Let’s take an example of XSS. The interviewer might first start with basic XSS questions like types of XSS, the difference between them, and so on. The more technical questions might include:

* If using CSP, how can you still allow some inline JavaScript to run?
* If using CSP nonce, how should the nonce be generated and set?
* Given a code snippet and some disallowed characters, is XSS possible in the scenario? Provide a payload.
* How does output sanitization help in mitigating XSS? Why is it better than input encoding?

Some other security concepts/issues I was asked about:

* Insecure deserialization (including providing an example in a language of choice).
* XML external entities (XXE).
* NoSQL Injection.
* Memory-based issues (buffer/heap overflow, use after free, double free).
* OAuth and its grant types.

## Code Review
The premise of the code review interview is to understand if by looking at code, you can identify security issues. I have been asked to review code at various stages of the interview process. Most often it is a part of the deeper technical interview. The basic technical interview can also sometimes include some basic code review. At one company, I had a dedicated code review interview. The things I had to look for (or rather, things I found) include XSS, SQL/NoSQL injection, authentication/authorization bypass, and missing security controls.

Not every code review was in a language I was familiar with, so you might have to read code in a language new to you. For the most part, they were still common languages with syntax I could more or less understand. Whenever I had a language I had not worked with, I informed the interviewer about it. They were happy to answer any language-specific questions I had, e.g. if I was unfamiliar with a certain function or certain way a statement was written. Languages I looked at include Python, Ruby, C, and Java. (One interviewer gave me a choice of Python, JavaScript, and a few others, I chose Python.)

## Threat Modeling
In nearly every company I interviewed with, I had to threat model a given system. Sometimes it was included in the deeper technical interview, other times it was a dedicated interview. I encountered two kinds of threat modeling questions:

* Abstract — I was asked to list potential threats in an application (eg: restaurant reservation system). This did not include any diagrams or workflows. My approach here was to list potential issues — and ask the interviewer if they wanted me to elaborate on anything. Based on what I listed, the interviewers asked me follow-up questions.
* Specific — I was given either architecture diagrams or sequence diagrams and sometimes implementation details and I had to find issues that exist in the given service. The service in question would be much smaller in scope as compared to the abstract case (eg: the service in the restaurant reservation system that takes input from restaurants and generates the listing for users). In this case, I could not just list potential issues in the service. I had to show how the malicious payload might enter the system, where it would be triggered, and what parts it would impact.

The interviewer might add additional scenarios or caveats to the service after the initial walkthrough. Some interviewers asked me to conduct the threat modeling as if I were working with a software engineer who was building the given service, and I was performing the security review. This meant I had to ask questions about the service or provide recommendations without using too much security jargon.

## Behavioral
Almost all companies had a behavioral round. Some had multiple rounds. I was asked general behavioral questions like:

* Can you give an example of a project or task that didn’t go as planned? What did you do to address the issues and what was the outcome?
* Tell me about a time when you had to adapt to a major change at work. How did you approach it?
* Give me an example of a time when you had to prioritize multiple tasks or projects. How did you decide what to focus on first?
* Describe a time when you had to learn a new skill or technology quickly. How did you go about it, and how did it impact your work?
* Can you share a situation where you faced a challenging problem at work? What steps did you take to analyze and solve it?
* Generally, these interviews were taken by someone not on the immediate team I was applying for.

## Coding
I have seen coding rounds being asked for roles that require building tooling or implementing security features. Since I was not looking for a role like that, most companies I applied to did not have a coding round. In the few that had, the questions were of two types:

* Leetcode easy/easy-medium type questions.
* General scripting questions that tested familiarity with a language of choice.

My understanding, from the explanations of the interviewers, is that they had a few projects that required some amount of coding so they wanted a candidate who was comfortable writing small scripts or building simple automated tools. They were not looking for a very proficient coder since coding was a small part of the responsibilities.

## Director / Leadership Conversations
More often than not, I had the opportunity to speak to the director of the team that was hiring. This was much later in the interview process, typically after I had completed the technical rounds. It was similar to the conversation with the manager — questions about my background and what I was looking for in my next role.

In a few rare instances, I had conversations with senior leadership including CTOs and/or CISOs. Although I was very nervous about speaking to senior leadership — they usually just wanted to get to know me and what I was looking for role / career-wise.

## Fun/Interesting questions
During my interviews, I was asked some fun/interesting questions. These don’t fall into any one interview in particular. They are not difficult questions, just things I was not prepared to answer or had never thought about — so they caught me slightly off guard.

* What is your favorite security vulnerability to work on?
* What is the last thing you learned? (The interviewer specified it did not have to be security or tech-related, I still spoke about some CVE…)
* What is something that you can teach me in the next 2 minutes (Again, the interviewer did not want a security or tech-related answer. If you know any party tricks, they will probably be a hit! I sadly did not know any.)
* Questions about my hobbies: It is possible that you and your interviewer share hobbies. I did, and I was asked about some specifics. Since I was not expecting a question about my hobbies, it took me a few moments to gather my thoughts.

***
I hope you found this useful. Do check out part 2, where I talk more about the interview prep. If you have any questions or need any help with interview prep, don’t hesitate to reach out to me (preferably on LinkedIn).