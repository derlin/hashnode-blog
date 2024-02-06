---
title: "Ever wondered how cloud providers (PaaS) integrate with GitHub? I did."
seoDescription: "Dissect with me how cloud providers integrate with GitHub, and deepen your knowledge of GitHub tooling (Actions, Web Apps, OAuth Apps)."
datePublished: Tue Feb 06 2024 13:00:39 GMT+0000 (Coordinated Universal Time)
cuid: clsadco6n00070al54db79e7n
slug: how-cloud-platforms-integrate-with-github
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1706461346463/40f02157-d836-4d6e-b51a-d32c435888e1.png
tags: cloud, github, learning, devops

---

With GitHub standing out as the leading platform for hosting public code, most cloud providers offer dedicated features for deploying code hosted on GitHub. I have always used them without much thought. That is, until recently.

Dissecting their inner workings, I found various approaches, some better than others. This gave me a deeper understanding of the GitHub tools available, how they work, and, well... I found this fascinating and wanted to share! So here it is.

---

* [The contenders](#heading-the-contenders)
    
* [Google Cloud Run: all the power of GitHub Apps](#heading-google-cloud-run-all-the-power-of-github-apps)
    
    * [Overview](#heading-overview)
        
    * [What is a GitHub App?](#heading-what-is-a-github-app)
        
    * [The process](#heading-the-process)
        
    * [Wait... What?](#heading-wait-what)
        
    * [Analysis](#heading-analysis)
        
* [Azure Static Web Apps: reversing the responsibility](#heading-azure-static-web-apps-reversing-the-responsibility)
    
    * [Overview](#heading-overview-1)
        
    * [What is a GitHub OAuth App?](#heading-what-is-a-github-oauth-app)
        
    * [What is a GitHub Actions workflow?](#heading-what-is-a-github-actions-workflow)
        
    * [The process](#heading-the-process-1)
        
    * [More about the Azure GitHub Action](#heading-more-about-the-azure-github-action)
        
    * [Analysis](#heading-analysis-1)
        
* [Heroku: keep it simple](#heading-heroku-keep-it-simple)
    
    * [Overview](#heading-overview-2)
        
    * [The process](#heading-the-process-2)
        
    * [Analysis](#heading-analysis-2)
        
* [Summary](#heading-summary)
    

---

## The contenders

Today, we look at [Google Cloud Run](https://cloud.google.com/run?hl=en), [Azure Static Web Apps](https://azure.microsoft.com/en-us/products/app-service/static), and [Heroku](https://www.heroku.com). They are not the sole cloud providers out there (not even the bests, check out [divio](https://docs.divio.com) 😉), but all three are relatively famous and have some sort of GitHub integration.

By GitHub integration, I mean the cloud provider has a dedicated process to link with a repository hosted on GitHub and provides additional services such as automatic deployments on push or preview apps on pull requests.

## Google Cloud Run: all the power of GitHub Apps

From their landing page, Google Cloud Run allows you to:

> Run frontend and backend services, batch jobs, deploy websites and applications, and queue processing workloads without the need to manage infrastructure.

I use Cloud Run to deploy my little [rickroller](https://github.com/derlin/rickroller) app.

### Overview

When setting up a new service in Cloud Run, you can choose to "*continuously deploy revisions from a source repository*". This supports BitBucket, Cloud Repositories, and of course, GitHub. When the latter is selected, here is what the wizard looks like:

![Google Cloud Run GitHub integration process](https://cdn.hashnode.com/res/hashnode/image/upload/v1705166514465/2f0812e8-9bf0-4773-b26c-31b5a5ea106e.png align="center")

There are two steps here, let's break them down.

### What is a GitHub App?

Cloud Run uses a [GitHub App](https://docs.github.com/en/apps/overview) (called [Google Cloud Build](https://github.com/apps/google-cloud-build)), which is a compelling concept. In short, GitHub Apps can act on your behalf, act on themselves (for example open issues), or act outside GitHub by reacting to events (via webhooks). Each App must declare the permissions it requires (e.g. only read repository contents), which determines the subset of API endpoints they can access.

Something fundamental is that GitHub Apps are ***installed at the organization level***. An App just needs to *exist* to impersonate you, but it needs to be *installed* to act on its own. When installing the app in an organization, it is up to you to define the list of repositories it can act on (all, or a selected few).

### The process

Back to Cloud Run. In the first step, Google tells you to *authenticate*. Here, it is using OAuth2 to [retrieve an access token to act on your behalf](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-with-a-github-app-on-behalf-of-a-user). This token will have the permissions both you AND the app have (impersonation).

Once authenticated, Google uses the token to ask the GitHub API about the *installations* of the GitHub App you have access to, and for each installation which repositories the installation can manage. This list is used to populate the second dropdown field.

The "*manage connected repositories*" redirects you to GitHub, where you can see your organizations and potentially install (or update the installation of) the App. After installing an app on an organization (and giving it access to some repos), Google updates the dropdown list displayed in the wizard.

Finally, once you select a repository in the list, Google saves somewhere the repository information and the corresponding installation ID of the App. With this, Google can [authenticate as a GitHub App installation](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/authenticating-as-a-github-app-installation) and start doing things. In this case, "doing things" is reacting to push events (via a webhook) to pull and redeploy your code on Cloud Run (and many other cool things that I won't list there). Its range of actions is limited by the set of permissions of the App.

### Wait... What?

Why all this complexity you ask? Let's break it down one more time:

* *Why the need for a user access token in the first step*? A GitHub App can list all its installations, but cannot determine which of those *you* have access to.
    
* *Why the need to install the app in the second step* (and not just use the user access token)? The user access token generated by a GitHub App has an expiration date. Google needs to perform actions without you being around to give your blessing. Only *installations* can request access tokens to act on themselves.
    

### Analysis

The way Google Cloud Run works may seem overly complicated, but there is currently no easier way to associate a GitHub App with a specific repository on GitHub (except by asking the user to enter manually the name of the repo and the installation ID, which is not user-friendly). Once Google associates a GitHub App Installation ID to the repository, it has all the power a GitHub App entails: it can do *anything*, being only limited by its permissions (which can be updated). This is a very stable and future-proof way of integrating with GitHub.

Another advantage of GitHub Apps is that they act on their own. If they publish a comment on a pull request or do a push, you will see the name of the App as the author. This allows for easy monitoring.

A user is free to opt out at any point by deleting the GitHub App installation.

## Azure Static Web Apps: reversing the responsibility

Azure Static Web Apps offers "*static content hosting and dynamic scale for integrated serverless APIs*".

### Overview

Contrary to Google, which implements a "pull" mechanism, Azure went for the "push" approach, taking advantage of GitHub OAuth Apps and GitHub Actions.

![Azure Static Web App GitHub integration process](https://cdn.hashnode.com/res/hashnode/image/upload/v1706440924513/285543c0-469c-4b43-b081-afd39471c0cd.png align="center")

### What is a GitHub OAuth App?

Unlike GitHub Apps, you do not install OAuth Apps because they can only *act on behalf of a user*. They cannot act on themselves as GitHub Apps can. It is not possible to limit the set of repositories an OAuth app can access, only their permissions (e.g. read vs read/write), and their access tokens never expire. There is no reason to use OAuth Apps nowadays, but they were the first historically, thus many integrations still use them.

To know more, read [Differences between GitHub Apps and OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/differences-between-github-apps-and-oauth-apps) (the GitHub documentation is amazing, btw).

### What is a GitHub Actions workflow?

From the [documentation](https://docs.github.com/en/actions/using-workflows/about-workflows),

> A workflow is a configurable automated process that will run one or more \[GitHub Actions\] jobs. Workflows are defined by a YAML file checked in to your repository and will run when triggered by an event in your repository, or they can be triggered manually, or at a defined schedule.

To paraphrase, workflows allow running automated tasks on GitHub (using runners) to implement CI (Continuous Integration, such as running tests) and CD (Continuous Deployment, such as building and publishing a Docker image or deploying a package to NPM).

Gitlab and others also have CI/CD. What is specific to GitHub is the notion of [GitHub Actions](https://docs.github.com/en/actions/quickstart). A workflow is composed of jobs that have a list of steps. A step can be a shell script or a call to an action. Think of an action as a way of packing complex logic into a simple component that can be reused, thus *extending* GitHub capabilities. Public Actions are stored on public GitHub repositories and may be published to the [MarketPlace](https://github.com/marketplace).

### The process

So, back to the wizard. First, Azure asks you to authenticate to GitHub (using the OAuth App), so it can act on your behalf. Next, it retrieves the list of your repositories and branches from the GitHub API to populate the dropdowns. Once you choose the target repository, it asks you to select a "build preset" (it currently supports frameworks such as React or Angular, and static website generators such as Gatsby or Hugo). Those presets are used to generate a GitHub Workflow file, which itself uses a GitHub Action maintained by Azure called [static-web-apps-deploy](https://github.com/Azure/static-web-apps-deploy).

When hitting deploy, Azure:

* commits the workflow file in the `.github/workflows` directory of your branch (with you as the author), and
    
* updates your repository settings to add environment secrets (under *Settings* &gt; *Secrets and Variables* &gt; *Actions*), such as the secret token used by the workflow to authenticate to Azure.
    

Note that this is a one-time action. Afterward, everything is done from GitHub through the magic of the Azure GitHub Action. In other words, it is the workflow running in a GitHub runner that calls an Azure API endpoint to deploy a new version. The responsibility is reversed.

Note that you don't need to use the wizard, you can also manually create the workflow file in your repo (and add the necessary secrets) and the result would be the same. I would even encourage you to go the manual way, to avoid granting Azure any permissions.

### More about the Azure GitHub Action

The default workflow committed by Azure can be seen here (with slight modifications depending on the preset): [https://learn.microsoft.com/en-us/azure/static-web-apps/build-configuration?tabs=github-actions#build-configuration](https://learn.microsoft.com/en-us/azure/static-web-apps/build-configuration?tabs=github-actions#build-configuration). Unfortunately, the GitHub Action, [Azure/static-web-apps-deploy](https://github.com/Azure/static-web-apps-deploy), hides its source code inside an executable from a Docker image. A shame, because I would love to analyze it!

With the default workflow, your Azure static web app is redeployed with the latest changes on every push to the main branch. Here is an output of the GitHub Action run:

```python
Zipping App Artifacts
Done Zipping App Artifacts
Uploading build artifacts.
Finished Upload. Polling on deployment.
Status: InProgress. Time: 0.0628832(s)
Status: Succeeded. Time: 15.1286488(s)
Deployment Complete :)
Visit your site at: https://blue-sea-084206610-1.centralus.4.azurestaticapps.net
Thanks for using Azure Static Web Apps!
```

Whenever a pull request targeting the main branch is opened, the action deploys instead a preview environment and adds a comment to the pull request with the preview URL. Every change on the pull request will update the preview environment. The preview environment is torn down when the pull request is closed (merged or discarded).

![Azure Action PR comment](https://cdn.hashnode.com/res/hashnode/image/upload/v1705684222828/9a6d1e13-78f7-477f-a968-da63517de183.png align="center")

It is possible to configure preview environments for other branches than `main` with a few changes to the GitHub Action's parameters.

And all of this works flawlessly! The only thing I had to change was the default permissions of the workflows (in *Settings* &gt; *actions* &gt; *workflow permissions*) from `read` to `read/write` to allow the Action to post comments on pull requests (see [this issue for details](https://github.com/Azure/static-web-apps/issues/797)).

### Analysis

I find the Azure approach brilliant.

Thanks to the GitHub Action, Azure itself doesn't need to know anything about git or GitHub (the content to deploy is sent as a zipped archive). There are no permissions involved since the workflow runs inside your repository. Reacting to pull requests is also easier since it is a built-in feature of workflows. To do the same with a GitHub App, you would require additional permissions, plus a server somewhere reacting to webhooks.

As a bonus, users can freely choose the version of the action they use, and parametrize it to their liking in a Configuration as Code (CAC) manner. There is no magic outside the repo itself. A GitHub Action is thus more transparent, and its power is only limited to the permissions of the `GITHUB_TOKEN` configured in the workflow.

The only drawback is the use of an OAuth App for automatic setup. OAuth App tokens remain active until they're revoked by the user, meaning Azure may still act on your behalf on all your private repositories after the setup unless you manually do something. I believe Azure discards the token after creating the workflow, but still. My two cents: revoke access to the Azure OAuth App after setup!

In short, the GitHub Action approach is less expensive, more sustainable, and user-friendly. Good job!

## Heroku: keep it simple

Heroku is a container-based cloud Platform as a Service (PaaS). Acquired in 2010 by Salesforce, it hosts your applications in AWS (without you having to know this fact).

### Overview

Heroku takes a simpler approach, combining a GitHub OAuth App and a repository webhook, while still offering advanced features such as preview apps (deployed on pull requests), automatic deployments, and more.

![Heroku GitHub integration process](https://cdn.hashnode.com/res/hashnode/image/upload/v1706440971720/289cb049-4450-4f06-bd44-b1d412afc560.png align="center")

### The process

To enable GitHub integration, Heroku asks you to allow the Heroku Dashboard OAuth application and give it *full control* over *all* your public and private repositories across *all* the organizations you are admin of. Because it uses an OAuth app, there are no fine-grained permissions: it has access to all or nothing.

Once authorized, you can search (no dropdown 😳) the name of a repository and "enable" it. Upon pressing enable, Heroku adds a webhook to your repository. From now on, every action on your repository will notify [https://kolkrabbi.heroku.com/hooks/github](https://kolkrabbi.heroku.com/hooks/github). The webhook is visible under your repository's *Settings* &gt; *Webhooks*.

With the webhook in place and the ability to act as you on GitHub (remember that OAuth user tokens do not expire), Heroku is now ready. In the Heroku settings, you can now enable automatic deployments, [Review Apps](https://devcenter.heroku.com/articles/github-integration-review-apps), and more.

In other words, when something happens on the repository, Heroku gets notified via the webhook and takes action depending on the Heroku settings. If it needs to interact with GitHub, it impersonates you. This is how you can see messages on pull requests such as "*&lt;your username&gt; deployed &lt;your app&gt; X seconds ago*".

### Analysis

This approach is efficient because, with only one action from you (the authorization of the App), Heroku can do *everything*. Adding new features is completely transparent because it listens to all events and asks for all the permissions *you* have on GitHub.

The downside? You are asked to trust Heroku completely and have no way of ensuring it doesn't abuse its power. A security breach such as [the one of April 2022](https://blog.heroku.com/april-2022-incident-review) (attackers accessed OAuth2 tokens from Heroku and used them to steal secrets stored in private GitHub repositories) can yield big consequences with unrestrained OAuth tokens. Remember: "*OAuth tokens remain active until they're revoked by the customer"*. It is up to Heroku to ensure it secures and rotates the tokens properly. Hopefully, they have competent engineers :).

To be fair, Heroku supported GitHub integrations before GitHub Apps appeared, and [is aware of the issue](https://devcenter.heroku.com/articles/github-integration#why-does-heroku-need-read-and-write-permissions-to-my-github-repos):

> GitHub Apps allow for finer grained control on repositories, but that has not been integrated into the GitHub integration yet. We cannot commit to a timeline but it’s definitely on our radar. Keep an eye on the Heroku Changelog.

Given their current implementation works perfectly, I doubt the migration to GitHub Apps will be prioritized soon (unless another security breach forces their hands).

## Summary

This article looked at three different ways for cloud platforms to interact with GitHub:

| Provider | Approach | Notes |
| --- | --- | --- |
| Google Cloud Run | Pull → GitHub App (+app webhook) | Limited permissions. Acts as you during setup, acts as itself otherwise. |
| Azure Static Web Apps | Push → GitHub Action | No permissions. Optional GitHub OAuth for setup only (theoretically one shot). |
| Heroku | Pull → GitHub OAuth App (+ repository webhook) | Full permissions, always acts as you. |

In my opinion, Azure is the clear winner with an original *push* approach. You are responsible for adding a workflow in your repository and you control what happens end-to-end. There is no hidden magic, and you are not required to give Azure any access to GitHub whatsoever, although you can choose to do so to benefit from a guided setup.

Google comes second, with the most correct way of implementing a pull approach as of 2023. GitHub Apps have fine-grained permissions and act as themselves, making their actions more transparent on GitHub. GitHub Apps are a bit more complex to manage though, with this whole authorization versus installation concept, and requires a very good wizard to make it simple for users. Google's wizard is not the best in class in this regard (I am not surprised).

Heroku's implementation based solely on a GitHub OAuth App is outdated and suffers from all the downsides pushing GitHub to ship GitHub Apps in the first place: lack of fine-grained permissions, Apps acting on behalf of (admin) users, no expiration of tokens, and more. It is however the easiest to put in place and maintain from a vendor perspective, and also the easiest for the user.

The main takeaways from this (already too long) article are:

* implementing a GitHub integration is not straightforward and must be planned carefully.
    
* GitHub is an amazing product with many points of extensions: GitHub Apps, GitHub OAuth Apps, and GitHub Actions are today the most prevalent.
    
* As a GitHub user, you may often be asked to authorize an extension. Learn how to read the prompts properly and always think of the security implications before saying yes.
    
* Don't forget to review permissions regularly, and revoke access to Apps you do not use anymore. On GitHub, click on your avatar on the top-right, *Settings* &gt; *Applications*.
    

---

I hope you enjoyed this article and learned something, let me know in the comments what you think and if you have a better integration approach undocumented here.