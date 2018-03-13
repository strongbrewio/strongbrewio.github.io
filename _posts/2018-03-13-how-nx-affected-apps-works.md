---
title: How does nx know which apps are affected
published: true
author: kwintenp
description: A deep dive into the implementation of how nx knows which apps are affected by a PR
layout: post
navigation: True
date:   2018-03-13
tags: Angular Nx
subclass: 'post'
categories: 'kwintenp'
logo: 'assets/images/strongbrewlogo.png'
published: true
disqus: true
cover: 'assets/images/cover/cover10.jpg'
---

Nx from Nrwl is a collection of tools that can help us build Angular applications using the monorepo approach used at google. In essense, Nx is a set of schematics that work on top of the @angular/cli. These schematics can be used to create apps and libs inside of a single @angular/cli project, which is supported by default. Nx leverages this feature and makes the process a litlle easier.

This post however is not about the basic working of Nx. For that, go to their official website <a href="https://nrwl.io/nx" target="_blank">here</a> or watch this <a target="_blank" href="https://www.youtube.com/watch?v=bMkKz8AedHc">very informative talk</a> by <a href="https://twitter.com/MrJamesHenry">James Henry</a> at NgVikings.

This post will cover a different aspect of Nx. When all your applications are in the same repository, this poses some problems for your CI environment. Whenever a PR is merged into the master, the CI environment must rebuild your applications and ideally publish them to your DEV environment. But how do you handle it if you have a lot of apps in the same monorepo and you only want to build the apps that were affected by a certain PR? Nx provides us with a script to handle this situation. 

## Problem description

Let's take a look at the following Nx workspace. It has 2 apps and 2 libs.

![nx-workspace-image](https://www.dropbox.com/s/4qohmskumvwa8k2/Screenshot%202018-03-13%2019.04.20.png?raw=1)

As you can see 'app1' depends on 'lib1' and 'lib2' and 'app2' only depends on 'lib1'. So if, in the case we changed something to 'lib2' we only need to rebuild our 'app1'. Nx provides us with a script that will, based on two git commit hashes, tell us all the apps that need to be build. In this post, we will take a look at the script they use to accomplish this.

-- insert the script --

## How do they know what apps to build

### Knowing what files are changed

To know the files that need to be changed, they leverage the 'git diff' command. Let's look at the code:

```typescript
function getFilesFromShash(sha1: string, sha2: string): string[] {
  return execSync(`git diff --name-only ${sha1} ${sha2}`)
    .toString('utf-8')
    .split('\n')
    .map(a => a.trim())
    .filter(a => a.length > 0);
}
```

The 'git diff' command returns all the files that have changed between two commits. This is transformed into a list of files.

### Identifying all the apps

Next step is identifying the different apps of the project. For this, they simply parse the '.angular-cli.json' file which has an entry with all the apps.

```typescript
export function getAffectedApps(touchedFiles: string[]): string[] {
  const config = JSON.parse(fs.readFileSync('.angular-cli.json', 'utf-8'));
  const projects = getProjectNodes(config);
  
  ...
} 

export function getProjectNodes(config) {
  return (config.apps ? config.apps : []).filter(p => p.name !== '$workspaceRoot').map(p => {
    return {
      name: p.name,
      root: p.root,
      type: p.root.startsWith('apps/') ? ProjectType.app : ProjectType.lib,
      files: allFilesInDir(path.dirname(p.root))
    };
  });
}
```
They fetch all the apps, loop over them, and create an object containing information about this app, the name, the root folder, is it a real App or a Lib and all the files it holds.










