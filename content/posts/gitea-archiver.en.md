---
title: "Gitea Archiver"
date: 2022-02-12T11:53:01+09:00
draft: false
description: "Script to transfer Gitea repositories to another account"
slug: "gitea-archiver"
---

There are many Git repositories being created every day to try out certain tutorials or to create prototypes of some kind. I used to push my code to a private Gitea to reuse the code in those repositories.

However, the project names often overlap, and the ad hoc repository names are scattered and inconsistent. For example, one day I might be pushing under the name `purplerain-2`, and another day it might be `purplerain-210212`, or `purplerain-0212`.

Since there were more than 100 such repositories on Gitea, I thought I could use Gitea's API to rename them, create an account called `archiver`, and search for the code via that account when I needed it.

## What I noticed when I wrote the code

Normally I start writing web applications before I even know what I'm going to use them for, but this time I started writing a script in JavaScript with the intention of throwing it away, but I wanted to treat Gitea responses as types, so I converted it to TypeScript in the middle.

What I learned this time was that **NPM packages written in ESM cannot be used as-is in TypeScript**, that you don't have to write everything when writing HTTP responses in Interface, and above all, that **it's better to try to write JSDoc in functions**.

### About the ESM package

This is not the first time I've had this problem, but I've been wondering whether programs written in JavaScript are for web browsers, Node.js, or both. And even in the same JavaScript, there are multiple versions such as ES5, ES2017, etc. It's troublesome to write programs with such versions in mind, so I thought that TypeScript like Webpack would absorb the whole thing, but that's not the case.

This is simply a lack of research, but the purpose of this project was not to deepen my knowledge of TS/JS, but to organize the Gitea repository, so I thought I'd leave it for another time.

### About Interface

I studied Java for a while, but I don't understand the difference between Java's Inteface and TypeScript's Interface that well. However, at the time, I understood the concept of Interface to be something like USB, which allows you to connect different machines together using a common standard.

If you trace a server's HTTP response with Interface, it will show up in the editor's completion function, and you'll feel like you're doing something, so I usually try to do my best, but Gitea's response is not that simple, so it might be a good way to practice expressing TypeScript, but I don't want to look back at the type file later.

```typescript
namespace Gitea {
  export interface User {
    login: string
  }

  export interface Repo {
    cloneUrl: string
    name: string
    owner: User
  }

  export interface CommitUser {
    name: string
    email: string
    date: string
  }

  export interface Commit {
    sha: string
    commit: {
      author: CommitUser
      committer: CommitUser
    }
  }
}
```

Gitea's response is documented in Swagger, so there were almost 100 lines of files created based on that information. But in the end, my interest this time was only the date and time needed to rename the repository, so I ended up with less than 30 lines. It's a natural amount to paste into a blog like this.

In other words, I think that Inteface is enough to run the TypeScript compiler.

### Be aware to write JSDoc

As I wrote some procedural functions in a program that grew up with the intention of writing them down, I decided that it would be better to write comments as I specified the types of arguments and return values because of TypeScript.

Originally, I didn't write comments or documentation in my programs at all, and even if I did, the most I could write was TODO comments. However, I didn't want to show it to anyone, and I didn't feel confident enough in English to write it myself, so I thought I wouldn't have to.

So I thought that writing JSDoc comments for each function with a minimum outline, arguments, and return values would be effective for clearing my head, which I've been doing a lot of programming in the beginning.

I don't have any particular plans to write it down as a document, but it's hard to practice something like test-driven programming when you don't usually write it, so I thought I'd try to keep it in mind to document it even if it's just a hobby.

## What I noticed when I tried to run it

Since I checked a series of processes while saving them in a JSON file, there were no cases this time where the program failed in the middle.

However, after running the program, there were cases where the renaming of repositories that originally contained timestamps was duplicated (e.g. `purplerain-20200212` became `purplerain-20200212-20200212`). This was caused by referring to the original name when referring to the renamed property, but since I was dealing with a server that was actually running, I was able to solve the problem by simply renaming it later, but I thought it would be difficult for a program that involves file manipulation.

It took me about a full day to do this work, and the output for this blog and README was done the next day. I achieved what I wanted to do in the beginning, and although I feel that I should publish this again, I thought it could be a product.

Translated with www.DeepL.com/Translator (free version)