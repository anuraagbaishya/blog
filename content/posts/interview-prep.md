---
title: "Tales from my Product Security Job Hunt: Part 2, Preparation"
date: 2024-01-14T21:51:34-08:00
toc: true
images:
tags:
  - interviews
---

This is part 2 of my interview experience and preparation series, where I will talk about how I prepared for the interviews. Check out part 1 where I talk more about the process and types of interviews [here](https://www.baishya.xyz/posts/types-of-interviews/)

As I mentioned in my previous post, this is catered to my search criteria, which was:

* Experience: Mid-level (~4–5 years of experience)
* Role: Product Security / Application Security

Even within the roles I was looking for, different openings had slightly different requirements. For example, an application security role for a team/company that has a large mobile application footprint is likely to involve duties related to securing these applications — and therefore, the candidate would be expected to know about mobile application security.

The responsibilities and therefore the desired skillset also varied with the size/maturity of the company’s security program. In smaller teams, the responsibilities were broader, whereas in larger teams, the responsibilities were more specialized. I mostly applied to roles with more specialized requirements, based on my interests and skillset. I did explore a few roles that had a broader scope, but since my experience and skills are quite specialized in application/product security, I did not fare well in these interviews — and I decided to look for roles where I would be a good fit.

Most of this post will be specific to roles having one or more of the following as the primary responsibilities:

* Understanding and triaging security issues including root cause analysis.
* Threat modeling and/or security reviews including code reviews.
* Building tooling/automation to detect application security issues.

## Technical Interviews
Most companies had multiple technical interviews, starting with basic technical questions and continuing to more in-depth or domain-specific questions. I spent most of my preparation time preparing for technical interviews. Having 4 years of experience in application security, and therefore being constantly exposed to topics in this space was a big help. My preparation in this area was mainly divided into:

* Topics that I do not work with (or rarely work with).
* Revising theoretical aspects of topics I worked with practically, e.g. TLS handshake or OAuth grants.

I primarily used the following resources:

* [**Portswigger Web Security Academy**](https://portswigger.net/web-security): This is by far my most used resource. The academy has detailed lessons on almost all topics you are likely to come across when applying for the type of roles I was looking at. The academy also has labs, which help in solidifying the theoretical understanding from going through the lessons. I concentrated on OWASP top 10/CWE top 25 and issues that are being discussed on the news/x.
* [**Web Security Questions by Tib3rius**](https://tib3rius.com/interview-questions): A great list of questions of all levels of difficulty.
* **News**: I generally read cybersecurity news and this habit helped me with interview prep. I was able to learn about security concepts that are being discussed actively or issues that adversaries are exploiting. I use a few sources for news:
  * [tl;dr sec newsletter](https://tldrsec.com/)
  * Blogs from security research organizations like [Palo Alto Networks Unit 42](https://unit42.paloaltonetworks.com/), [Google Project Zero](https://googleprojectzero.blogspot.com/), [Assetnote](https://blog.assetnote.io/), [Trend Micro ZDI](https://www.zerodayinitiative.com/blog), [Cisco Talos Blog](https://blog.talosintelligence.com/), etc.
  * Social Media (LinkedIn/X)
  * Medium blogs

* **_Unconventional_ Sources**: I also find information related to security using some of these non-conventional sources:
  * YouTube: Many hackers/security experts make educational videos on a variety of security-related topics. Some of my favorites are: [IppSec](https://www.youtube.com/@ippsec), [LiveOverflow](https://www.youtube.com/@LiveOverflow), [John Hammond](https://www.youtube.com/@_JohnHammond), [Bug Bounty Reports Explained](https://www.youtube.com/@BugBountyReportsExplained), [NahamSec](https://www.youtube.com/@NahamSec)
  * [HackerOne hacktivity](https://hackerone.com/hacktivity/overview): Reading through bug bounty disclosures can provide insights into possible ways to exploit vulnerabilities. Some of the reports are very creative.

**Books**: Although I did not use these books specifically for interview preparation, I have learned a lot from them.
  * The Tangled Web — Michal Zalewski
  * Computer Security, A Hands-on Approach — Wenliang Du
  * The Web Application Hackers Handbook — Dafydd Stuttard and Marcus Pinto
  * Cryptography and Network Security — William Stallings
  * The Hacker Playbook — Peter Kim

**Practical Exploitation Practice**: I feel I learn better by doing things. So I try my hand at CTFs or platforms like HackTheBox, TryHackMe, PortSwigger Web Security Academy, and others. I did not do these specifically for the interviews, it is just something I do sometimes to sharpen my skills.

## Code Review
Due to my interest in static analysis, I looked at a lot of code for security flaws at work. This prepared me for the code review interviews. Here are some tips that might be helpful:

* Have a checklist of things to look for. [This](https://github.com/mgreiler/secure-code-review-checklist) GitHub repo provides a great list that you can adapt for yourself.
* Practice with some known vulnerable apps. There are many to choose from:
  * [Juice Shop](https://github.com/juice-shop/juice-shop)
  * [DVWA](https://github.com/digininja/DVWA)
  * [WebGoat](https://github.com/WebGoat/WebGoat)
  * Many more are listed [here](https://github.com/vavkamil/awesome-vulnerable-apps)

* Semgrep Playground has an option to try out existing rules. The rule test has code that Semgrep will flag, and code it will not. Assuming the rule is correct, the code that Semgrep will flag will be vulnerable, and therefore you can use that to understand vulnerable patterns.

## Threat Modeling Interview
To be good at threat modeling, you will first need to have good technical knowledge. If you know how certain issues can impact systems, when you see a system that has these deficiencies, you can identify the issues. The next step would be to identify a threat modeling process that works best for you. There is no best method, but in my experience the most popular is STRIDE. There are other frameworks too like DREAD or PASTA. You can practice by looking at architectures and enumerating threats in them. You can also take an application (eg: a restaurant reservation system) and list potential threats in such an application.

Here are some resources to prepare for threat modeling interviews:

* [The Ultimate Beginner’s Guide to Threat Modeling](https://shostack.org/resources/threat-modeling)
* [Threat Modeling Process](https://owasp.org/www-community/Threat_Modeling_Process)
* [List of example STRIDE threats](https://threat-modeling.com/the-ultimate-list-of-stride-threat-examples/)
* [Shostack + Associates “Threat Model Thursday” Series](https://shostack.org/blog/category/threat-model-thursday)

## Coding Interview
I did not prepare much for coding interviews since the roles I was interested in did not have a coding interview. I relied on my coding skills from work or leisure projects to attempt the very few coding interviews I had. There are already many resources on the internet that provide detailed preparation guides for coding interviews. Please refer to them for coding interview guidance.

## Behavioral Interview
Preparing for a behavioral interview involves understanding the format and anticipating questions that assess your past behavior in certain situations. It is important to have personal examples ready for commonly asked behavioral questions. There are many lists of common behavioral questions that you can look at, or you can ask ChatGPT to provide some examples. Using the STAR method is a great way to answer behavioral questions. STAR stands for Situation, Task, Action, and Result. Here’s an example (courtesy of ChatGPT):

*Can you provide an example of a time when you had to overcome a significant challenge in a team project?*

*Situation*: In my previous role as a project manager, our team was tasked with implementing a new software system within a tight deadline.

*Task*: One of the key challenges was that we faced unexpected technical issues during the implementation phase, jeopardizing the project timeline.

*Action*: Recognizing the urgency, I immediately called for a team meeting to discuss the issues openly. I assigned specific tasks to team members based on their expertise, ensuring everyone had a clear role in addressing the technical challenges. Simultaneously, I collaborated with the IT department to expedite troubleshooting.

*Result*: Through our collaborative efforts, we successfully identified and resolved the technical issues within the deadline. This experience highlighted the importance of effective communication and swift problem-solving. As a result, our team developed stronger bonds and increased our overall efficiency in handling unexpected challenges.

## Leadership Interviews
I am clubbing interviews with the manager, director, or more senior leadership in this section. In general, I felt these were more conversations than interviews — to understand my experiences and interests and to tell me more about the role or the company. Be prepared to answer questions about your experiences and background, why you are looking for a change, and what interests you in the company. It helps to familiarize yourself a little with the products or services the company offers. There might be some technical questions involved. I have already covered above how you can prepare for these.

## General Tips
Some general tips that might help with the preparation or interview:

* Read the job description thoroughly. The technical interviews were almost always related to the responsibilities mentioned in the job description.
* Ask the recruiter about the interview process — how many rounds will be there, what each round will entail, how long you can expect the process to take, and if they have any preparation tips.
* Ask questions to your interviewers. Some examples: what challenges they face, what tasks/projects they find most interesting, why they enjoy working at the company, and so on. If you are speaking with the manager or leadership, ask them about their leadership style, what specific tasks you might work on, how they promote employee growth, and so on.
* Be honest. If you don’t know something, let your interviewer know about it. Take a guess based on what you know — but be sure to inform your interviewer.
* Try to keep calm. Interviews can be nerve-wracking, especially if you feel they are not going well. Even if you are not able to answer anything, keeping calm will let you think much better and take educated guesses. The interviewers might not be expecting you to know everything — and they might want to understand how you think or approach problems.
* Try to free up your schedule at work. Since panel interviews will likely be multiple interviews within a day or two, you will need to have a few hours free on various days. Therefore, plan your work ahead of time. Make sure once you are actively interviewing, you have a good number of available slots.
* Self-analyze your performance after an interview. Determine what areas you did well on and what areas you need to spend more time preparing. The first few interviews for me did not go very well — because I did not know what to expect in terms of questions, and also how well I had prepared. From them, I was able to prepare better for the upcoming interviews.