---
title: "Summary"
teaching: 14
exercises: 0
math: true

questions:
- "How can I demonstrate my awesome cluster configuration?"

objectives:
- "Start a log of configuration steps and performance."
- "Organize team investigations using the scientific method."

keypoints:
- "Cluster configuration has reproducibility challenges."
- "Spend extra time to plan well-defined performance tests."
---

## TL; DR;

A lot of what we communicate has an answer-bias.
However, trying to jump directly to the final answer
makes it hard to learn *why* the answer works.

As a competition team, it's extremely important to
keep an organized record of your efforts.  This helps
both to keep team members informed about what everyone
is working on, and also to document your team's activities
for writing your project report.

## Implement Kanban

Watch the "Agile Methodologies" presentation and
follow the associated hands-on tutorial section
of the [BSSW Tutorial](https://bssw-tutorial.github.io).

> ## Setting up Kanban
>
> Spend some time setting up a team Kanban board.
> Either Trello or a project board on github will
> work just as well.
>
{: .challenge}

> ## Create an Initial Issue List
>
> By now, your team should have a lot of ideas to discuss:
> virtual machine setup, networking and hardware configuration,
> test benchmarks, and ideas for tweaking performance.
> Everyone should write down a TODO list locally,
> and bring it for a team discussion.  As a team,
> you should interactively create a few tasks and assign them
> to the most energetic and enthusiastic members.
{: .challenge}

> ## Intermezzo
>
> Watch [12 Ways to Fool the Masses with Irreproducible Results](https://www.youtube.com/watch?v=R2-GuH-6VFU)
> for a fun look into the history of HPC performance reporting.
> You should come away from the video with a good understanding
> of the target audience for your report.
> The emphasis on documenting software versions, compile flags,
> and extra information needed for reproducing your work
> is especially helpful for doing a self-review on your work.
{: .challenge}

## Follow-Up

Now your team is off to a great start.  As you
work through the competition, try and do the following:

1. Keep creating new issues for work you don't have time to address at the moment.

2. Post status updates to the issues you are working on.  Don't be afraid to modify their descriptions to reflect what you're really doing, or close issues with short explanations of why they are resolved (or, alternately, no longer important).

3. Discuss recently completed tasks at regular team meetings.  Every completed task should have a paragraph or two of text explaining why the task is important, what you did, and what results you obtained.

4. Maintain a separate document with all these paragraph descriptions.  Present this as part of your final report.


## Reporting

The scientific method contains hypothesis generation, carrying out
experiments, collecting data, and writing up results.
Your progress on HPL and HPCG benchmarks will be graded based
on the optimization ideas you generated and tested.
The final report should give some context for the HPL/HPCG benchmarks
and system hardware.  Then have an organized discussion of
the key steps you took to arrive at your final configuration.
Things like initial "unoptimized" performance, steps to
setup MPI, select blas libraries, threading, job launch, and NUMA
configuration, and tile-size optimization should be reported.
You may choose to write these as a list of trials with with
separate "initial" and "final" timings
for each or as a sequential improvement after each configuration
change.

Every controlled test is important!  Even tests that result
in decreased performance will contribute well to your overall
evaluation as long as they are well explained and the
results are documented.

The report should have a table listing the compuational
throughput ("GFLOPS") reported by HPL and HPCG benchmarks
for key steps along your configuration journey.  It should
also include "theoretical peak GFLOPS" estimated based on
the hardware documentation.


{% include links.md %}
