---
layout: post
title:  "Caching node_modules in Azure DevOps CI pipelines"
date:   2020-09-16 09:00:00 +0100
categories: [DevOps]
comments: true
excerpt: "In this article we'll be taking a look at how to speed up builds in CI pipelines of Azure Devops by caching the node_modules folder..."
---

In this article we'll be using the Cache task of the Azure DevOps CI pipeline to cache the node_modules folder.

CI/CD is a big topic but for the purpose of this artice here is a brief summary:

#### CI
CI is when developer working copies are merged to a shared master branch of code in a source code system several times a day, automatically triggering what is called a build. A build is the process of automatically producing all the artifacts that are required to successfully deploy, test, and run production software. The benefit of CI is that it automatically gives the team feedback on the quality of the changes that are being made.

#### CD
CD is the process of getting changes that developers make to the software into production, regularly and safely, in a sustainable way. So, it is the process of taking the build from CI and getting that deployed to the production environment. The CI build may be deployed to a staging environment where the end-to-end tests are executed and passed before deployment is made to the production environment. At its most extreme, the CD is fully automated and triggered when a CI build finishes. Often, a member of the team has to approve the final step of deploying the software to production, which should have already passed a series of automated tests in staging. CD is also not always triggered automatically when a CI build finishes; sometimes, it is automatically triggered at a particular time of day. The benefit of CD is that the development team deliver value to the users of the software faster and more reliably.

#### Why cache node_modules?
As you may know the node_modules folder can potentially be huge (up to gigabytes) containing hundreds of node packages required by the front end application (whether that be an Angular, React or plain JavaScript application). During the build stage this folder needs to be present in order to build the application, and publish the relevant files for staging/production, this is done using commands such as `npm install` or `yarn install` and can take from a few to many minutes to download from npm (up to 40-50 minutes for bigger apps).

This might not be a problem in some cases or when integration and releases are less frequent, but in most situations it can be a bottleneck especially if the team wants to release frequently or push out a hot fix to to resolve a live issue.

#### Azure DevOps Cache task
In order to speed this up we can use the Cache task in the Azure DevOps pipeline, the task is made up of a *Key* and a *Path* which is basically a key/value pair used in many cache solutions along with a Cache hit variable used to determine if the cache was used (hit).

##### Key
The value in key is used to determine if the cache should be restored, the first time the task is run the key will be new and so there will be nothing in cache for it. You can enter a string value for the key or a path to file(s). In our scenario we are going to enter the path to the package.json file, so the key will remain the same (and thus use the cache) until it is updated when new packages are added. This is a handy way to let the pipeline take care of when to use cache.

##### Path
The path is the path to the file/folder that you want to be cached, this can be a pipeline variable that has a path value or you can enter the path directly. In our case we will enter the path to the node_modules folder.

##### Cache hit variable
The cache hit variable is set to true when the cache is used, this will be false the first time the task is run.


#### Adding / configuring node_modules caching
##### Cache task
- Navigate to the CI pipeline and click Edit
- Add a new task and search for Cache 
- Give the task a meaningful name such as "Restore cached node_modules"
- In the Key section add the path to your package.json file eg: `$(Build.SourcesDirectory)/myApp/fe/package.json`
- In the Cache hit variable enter a name for the variable such as `NODE_MODULES_RESTORED`

$(Build.SourcesDirectory) contains the path to the directory on the VM where the app files are located.
You can also add $(Agent.OS) to the start of the key because the agent may change across different builds and thus the cache would be located on a different VM. 

##### Bash task
- Add a Bash task and name it Yarn Insall (or something similar)
- Set the Type to inline and enter `yarn --cwd <path to folder where package.json resides> install` 
- Expand Control Options and select Custom conditions under Run this task and enter: `ne(variables.NODE_MODULES_RESTORED, 'true')`

the --cwd option on the yarn command sets the current working directory which needs to be the path to where your package.json file is located 

The Bash task will only be run if the cache hit `NODE_MODULES_RESTORED` variable is not equal to `true` (and thus the cache has not been restored)
This will be the case the very first time the pipeline is run with the two new tasks added.

You'll also notice a *Post-job: Restore cached node_modules* task will automatically be added to the pipeline, this update the cache, and in our case adds the downloaded node_modules to the cache.

The second time the pipeline is run the cache will have the node_modules that were saved by the post-job and the `NODE_MODULES_RESTORED` variable will be set to `true` so the Bash task will be skipped. 






