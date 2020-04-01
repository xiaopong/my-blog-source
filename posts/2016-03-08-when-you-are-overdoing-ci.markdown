---
title:  When you are overdoing continuous integration
author: xp
---
I was having a coffee with a couple of friends last Saturday, and one of them said that since they started doing continuous integration (CI), he has never been busier in project building and tools making. On top of the project works he has to do, he now spends a lot of time writing Jenkins plugins, integrating, debugging, and configuring. After five months of doing CI, he came to the conclusion that something is wrong.

The question was: what's wrong?

Like any methodology or paradigm that people get acquainted with, they tend to think of it as a panacea. And this is a case where people think of CI as panacea to all evils, and start to overdo it.

First of all, let's just have a common understanding that, continuous integration might be a buzzword du jour, but ultimately, it's just a process to make your project run more smoothly. Regardless of the tools you use, be it Jenkins, Continuum, BuildBot, Strider or what not, they are just tools. They are the means to achieve the end. What is important is the project itself, and that's the ball you need to keep your eyes on.

Now, let's see when do you know you are overdoing it.

- You spend more time working on the tools than working on the project, assuming that you are not into tool making business. In that case, you have to re-think about your process. Either the tool is too immature, too complicated, or it is a misfit for your project.
- Say, you are using Jenkins, and you have installed hundreds of plugins, and yet, you still need to write more custom plugins. When something is too complicated and too bloated, that's a sign you need to step back and re-think.
- You need a dedicated team to work on and maintain the tools, again, assuming that you are not into tool making business. Although, traditionally,  it is quite normal that a large software project requires a team for software configuration management (SCM). However, the gist of CI in an agile DevOps environment is certainly not to maintain a large SCM team.
- Each job is too big, and is not broken down into smaller jobs. Whether you like it or not, big jobs tend to make Jenkins (or any other tools) complicated. In that case, it is probably better to refactor the code base into more manageable pieces first, or refactor the build workflow, instead of overworking on the build tools.

Ultimately, continuous integration is a practice. It is about what you do, and not about the tools you use. You don't need all these fancy frameworks to really do CI. You might just have a few scripts and a couple of cron jobs, yet you might still be practicing continuous integration. CI is about splitting changes into small increments, and have the discipline to integrate frequently to not break the build.

CI is about behavior and mentality, do not fall into the trap of thinking that your team is practicing CI just because you have all the tools set up and running. And if you must constantly come back to work on the tools, instead of working on your project, you are overdoing it.
