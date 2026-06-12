---
title: Conventions
date: 2024-12-17
categories: [programming, git]
tags: [PR, code]
---

I wanted a central place where I could look up these details as I learn them and also point others too that ask, Where I have borrowed the content for my own use, I'll include the link in the section.

# Code review comments

See https://conventionalcomments.org/ for the original content

<!-- more -->

## Format

\<label> [decorations]: \<subject>

[discussion]

suggest using the following labels:

| Label | Description |
|---|---|
| **praise**:	|Praises highlight something positive. Try to leave at least one of these comments per review. Do not leave false praise (which can actually be damaging). Do look for something to sincerely praise.|
| **nitpick**:	|Nitpicks are trivial preference-based requests. These should be non-blocking by nature.|
|**suggestion**:	|Suggestions propose improvements to the current subject. It's important to be explicit and clear on what is being suggested and why it is an improvement. Consider using patches and the blocking or non-blocking decorations to further communicate your intent.|
|**issue**:	|Issues highlight specific problems with the subject under review. These problems can be user-facing or behind the scenes. It is strongly recommended to pair this comment with a suggestion. If you are not sure if a problem exists or not, consider leaving a question.|
|**todo**:	|TODO's are small, trivial, but necessary changes. Distinguishing todo comments from issues: or suggestions: helps direct the reader's attention to comments requiring more involvement.|
| **question**:	|Questions are appropriate if you have a potential concern but are not quite sure if it's relevant or not. Asking the author for clarification or investigation can lead to a quick resolution.|
| **thought**:	|Thoughts represent an idea that popped up from reviewing. These comments are non-blocking by nature, but they are extremely valuable and can lead to more focused initiatives and mentoring opportunities.|
| **chore**:	|Chores are simple tasks that must be done before the subject can be “officially” accepted. Usually, these comments reference some common process. Try to leave a link to the process description so that the reader knows how to resolve the chore.|
| **note**:	|Notes are always non-blocking and simply highlight something the reader should take note of.|
