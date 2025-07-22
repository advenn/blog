---
date: '2025-07-22T22:39:19+05:00'
draft: false
title: 'Apps Script'
tags: ["apps-script", "automation"]
---

## Beginning
I mainly started freelancing from 2024. At that time I had a client who had accounting/consulting firm, they needed a telegram bot to create template word contracts.
They said they had started doing it but didn't have time to finish it themselves. I saw their code, it was on apps script. At that time I first came across with apps script. I made the telegram with aiogram/python. 
btw, I used encoding files with base64, docxtpl to render docx files from template.

## Fully employment of Apps Script platform lately
2 weeks ago when I finished one of my freelance project with logistics company, my university mate, who runs a business for american market, asked automation of their reports.
I learnt how they do their business, the main business messages were run on messenger, on groups. So the task was to use telegram bot to carry data. 
I remembered about apps script, I still has access to that client scripts, researched how they did their telegram automation on apps script, besides that I employed, of course, LLMs.
Learnt how telegram bot can be run on apps script, and started the work. I set up webhook handler, formed a format to send message to database through telegram bot. 
I chose google sheets as database:
* it is accessible to customers without any additional tools.
* it is easy to manipulate those data
* and mainly it is free)

After the data source is fixed, I started working over the main thing on this job, the one page report sheet. Discussing with my mate we created a template, and I started working over the report.
I must admit it was pain in the *ss. i use python and go on work, i little have experience with vanilla JS, which apps script uses.
I had a lot of difficulties with timezones. I save them with uni timestamp and EST timezone date strings. On filtering i must parse the request on EST time, but it was difficult to understand how js parsing the time objects.
After all I fixed all the problems, I used LLMs' help of course, mainly Claude, Gemini abd Deepseek.
I handed the program for check, test. Customer asked additions, some format change, all of them were done and the last response to my work is: ![this](/photos/response-AK.png)

## Conclusion
Apps Script is a low code, serverless platform by Google, and works great with Google workspace apps, like google sheets, docs, and others. It has all the APIs for them and integrated well.
It has external HTTP interface that it can easily can receive GET/POST requests and can easily serve as a server via its UrlFetchApp interface.