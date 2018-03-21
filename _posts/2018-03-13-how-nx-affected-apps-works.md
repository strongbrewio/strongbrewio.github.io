---
title: Understanding how Nx identifies updated apps
published: true
author: kwintenp
description: A deep dive into the implementation of how Nx knows which apps are affected by a PR
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

Nx from Nrwl is a collection of tools that can help us build Angular applications using a monorepo and all the benefits that brings. In essence, Nx is a set of schematics that work on top of the @angular/cli. These schematics can be used to create apps and libs inside of a single @angular/cli project. Having multiple apps is supported by default and Nx leverages this feature and makes the process a little easier.

This post however is not about the basic working of Nx. For that, go to their official website <a href="https://nrwl.io/nx" target="_blank">here</a> or watch this <a target="_blank" href="https://www.youtube.com/watch?v=bMkKz8AedHc">very informative talk</a> by <a href="https://twitter.com/MrJamesHenry">James Henry</a> at NgVikings.

This post will cover a different aspect of Nx. When all your applications are in the same repository, this poses some problems for your CI environment. Whenever a PR is merged into the master, the CI environment must rebuild your applications and ideally publish them to your DEV environment. But how do you handle it if you have a lot of apps in the same monorepo and you only want to build the apps that were affected by a certain PR? Nx provides us with a script to handle this situation. 

## Problem description

Let's take a look at the following Nx workspace. It has 2 apps and 2 libs.

![nx-workspace-image](https://www.dropbox.com/s/4qohmskumvwa8k2/Screenshot%202018-03-13%2019.04.20.png?raw=1)

As you can see 'app1' depends on 'lib1' and 'lib2' and 'app2' only depends on 'lib1'. When we change something to 'lib2' we only need to rebuild our 'app1'. Nx provides us with a script that will, based on two git commit hashes, tell us all the apps that need to be build. You can run the script like this:

```bash
./node_modules/.bin/nx affected apps SHA1 SHA2
```
SHA1 is the previous commit hash and SHA2 is the next commit hash. If, in our example, we change something to lib2, this script will output:

```typescript
app1
```

In this post, we will take a look at the script you can use for this and look at some code snippets to understand how it works.

## How do they know what apps to build

### Knowing what files are changed

To know the files that have changed, they leverage the `git diff` command. Let's look at the code:

```typescript
function getFilesFromShash(sha1: string, sha2: string): string[] {
  return execSync(`git diff --name-only ${sha1} ${sha2}`)
    .toString('utf-8')
    .split('\n')
    .map(a => a.trim())
    .filter(a => a.length > 0);
}
```

The `git diff` command returns all the files that have changed between two commits. This is transformed into a list of files.

### Knowing the apps these touched files belong to

Now it is clear which files are touched, it's time to identify the 'projects' these files belong to. In the script, both Nx apps and libs are referenced to as projects. They implement this using the following code.

```typescript
export function touchedProjects(projects: ProjectNode[], touchedFiles: string[]) {
  projects = normalizeProjects(projects);
  touchedFiles = normalizeFiles(touchedFiles);
  return touchedFiles.map(f => {
    const p = projects.filter(project => project.files.indexOf(f) > -1)[0];
    return p ? p.name : null;
  });
}
```
This will give us all the projects that have files that have changed, a.k.a. the 'touchedProjects'.

### Identifying all the apps

Next step is identifying the different apps of the project. For this, they simply parse the '.angular-cli.json' file which has an entry with all the apps. Notice that 'apps' in this context means entries in the '.angular-cli.json'. Both the 'apps' and 'libs' in Nx terminology are 'apps' in the '.angular-cli.json'.

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
They fetch all the apps, loop over them, and create an object containing information about this app:
- the name
- the root folder 
- is it a real App or a Lib
- all the files it holds.

### Knowing the dependencies of the apps and libs

Now that the apps are identified and we know the files inside those apps, it's time to identify the dependencies the apps have on the different libs in the Nx workspace. To do that, they loop over every file in every project and parse those files using typescript. Then they visit every typescript node and if they encounter an import declaration or a `loadChildren` property they call the `addDeppIfNeeded` method since these are indicators that we might have a dependency on a lib in the monorepo.


```typescript
 private processAllFiles() {
    this.projects.forEach(p => {
      p.files.forEach(f => {
        this.processFile(p.name, f);
      });
    });
  }

  private processFile(projectName: string, filePath: string): void {
    if (path.extname(filePath) === '.ts') {
      const tsFile = ts.createSourceFile(filePath, this.fileRead(filePath), ts.ScriptTarget.Latest, true);
      this.processNode(projectName, tsFile);
    }
  }
  
private processNode(projectName: string, node: ts.Node): void {
    if (node.kind === ts.SyntaxKind.ImportDeclaration) {
      const imp = this.getStringLiteralValue((node as ts.ImportDeclaration).moduleSpecifier);
      this.addDepIfNeeded(imp, projectName, DependencyType.es6Import);
      return; // stop traversing downwards
    }

    if (node.kind === ts.SyntaxKind.PropertyAssignment) {
      const name = this.getPropertyAssignmentName((node as ts.PropertyAssignment).name);
      if (name === 'loadChildren') {
        const init = (node as ts.PropertyAssignment).initializer;
        if (init.kind === ts.SyntaxKind.StringLiteral) {
          const childrenExpr = this.getStringLiteralValue(init);
          this.addDepIfNeeded(childrenExpr, projectName, DependencyType.loadChildren);
          return; // stop traversing downwards
        }
      }
    }
    /**
     * Continue traversing down the AST from the current node
     */
    ts.forEachChild(node, child => this.processNode(projectName, child));
  }  
```
Let's look at the 'addDepIfNeeded' method.

```typescript 
 private addDepIfNeeded(expr: string, projectName: string, depType: DependencyType) {
    const matchingProject = this.projectNames.filter(
      a =>
        expr === `@${this.npmScope}/${a}` ||
        expr.startsWith(`@${this.npmScope}/${a}#`) ||
        expr.startsWith(`@${this.npmScope}/${a}/`)
    )[0];

    if (matchingProject) {
      this.deps[projectName].push({projectName: matchingProject, type: depType});
    }
  }
```

This method checks if the `loadChildren` property or the import declaration is linked to one of our own libs. In that case, we add the dependency to the list of dependencies per project using the 'projectName' identifier.

### Putting the pieces together

We found the files that were changed and to which projects they belong to. We figured out all the dependencies the different projects have. Now it just a matter of cross referencing these to figure out which 'apps' need to be rebuild. Let's look at the code:

```typescript
  if (tp.indexOf(null) > -1) {
    return projects.filter(p => p.type === ProjectType.app).map(p => p.name);
  } else {
    return projects
    			.filter(p => p.type === ProjectType.app)
    			.map(p => p.name)
    			.filter(name => hasDependencyOnTouchedProjects(name, tp, deps, []));
  }
```

There is an interesting part in this snippet. There is a check to see if 'null' is in the 'touchedProjects'. This happens when there is a change to a file outside of the 'apps' or 'libs' directory. This can happen if, for example, the package.json file has been updated. In that case, every 'app' needs to be rebuild.

Finally, we can look at the `hasDependencyOnTouchedProjects` function.

```typescript
function hasDependencyOnTouchedProjects(project: string, touchedProjects: string[], deps: { [projectName: string]: Dependency[] }, visisted: string[]) {
  if (touchedProjects.indexOf(project) > -1) return true;
  if (visisted.indexOf(project) > -1) return false;
  return deps[project]
  	.map(d => d.projectName)
  	.filter(k => 
  		hasDependencyOnTouchedProjects(
  			k, 
  			touchedProjects, 
  			deps, 
  			[...visisted, project]
  		)
  	).length > 0;
}
```

Using recursion, they cross-reference the affected files and the projects they belong to with the dependencies all the projects have. This will return a list of all the apps that need to build.

### Conclusion

Using git, typescript and a little javascript code, the guys at Nx created a script that can help us to only rebuild apps that are affected by a PR reducing the build time on our CI environments.







