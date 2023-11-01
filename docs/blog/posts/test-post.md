---
date: 2023-11-01  
authors: [allann]
categories:
  - minimal api
draft: true
tags: [design]
---

# Project Directory Structures

Various methods exist for organizing a project's directory structure, but I have a particular preference. Prior to delving into that, let's examine the conventional approaches often suggested by templates and demoware and why they may not be the most advantageous choice.

## Organization by Technical Concern

To illustrate this point, consider the analogy of a library—a repository of books available for reading, borrowing, and taking home. In a library, books are methodically arranged on shelves according to their respective categories. This systematic arrangement simplifies the process of finding a specific book and, at the same time, enables the discovery of related literature.

Similarly, in the realm of software development, projects are frequently organized based on **technical considerations**. This methodology entails the creation of distinct categories such as `Repositories`, `Services`, `Validators`, and `Mappers` to classify and manage various project components.

However, this approach can present challenges in terms of comprehending and navigating the codebase, particularly for those who are unfamiliar with it. The task of locating code associated with specific features becomes notably convoluted, leading to an increased cognitive load when implementing updates, conducting refactoring, or simply navigating the codebase.

It is worth noting that when components are organized in this fashion, they often encompass a wide range of functionalities. For instance, a Repository or a Service may contain code relevant to multiple features.

### The MVC Conundrum

Certain template structures inherently foster the segregation of technological components, further exacerbating the aforementioned challenges. A prime exemplar of this is the Model-View-Controller (MVC) architectural pattern.

When embarking on the development of a new MVC-based API, it is customary to create dedicated folders for Controllers, Models, and Views. This inherent design encourages the segregation of components based on their technological function once more, perpetuating the navigational and cognitive hurdles.

## My Preferred Approach

While the library's category system simplifies finding and accessing books, it may not be the optimal model for source code organization, in my view.

In the context of source code, a single feature often spans multiple files, each with distinct responsibilities or categories. When we access code for maintenance, we generally do so within the context of a **feature**, or more precisely, within a narrow vertical slice. So, how should we organize our code instead? The rule is straightforward.

!!! tip ""
    Organize by **action/utility**.

What does this entail?

- Deliberate consideration of how the code will be used, accessed, and maintained.
- Thoughtful examination of which code components will naturally evolve in tandem.
- The conscious effort to reduce the cognitive load associated with future actions.

In essence, this encapsulates the concept of Feature Folders. Additionally, this approach can be further refined through the combined utilization of Area and Feature folders.