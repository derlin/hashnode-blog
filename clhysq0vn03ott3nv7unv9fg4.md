---
title: "A developer's guide to deploy applications on Kubernetes"
seoDescription: "Learn about Kubernetes 101, manifests, Helm, helmfile, and Argo CD from a developer's perspective to deploy your apps on k8s with style."
datePublished: Mon May 22 2023 12:00:39 GMT+0000 (Coordinated Universal Time)
cuid: clhysq0vn03ott3nv7unv9fg4
slug: a-developers-guide-to-deploy-applications-on-kubernetes
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1683970495680/969b62b7-c5d2-4b8c-9f54-4b5aafad8b03.png
tags: tutorial, kubernetes, developer, helm, helmfile

---

If you haven't heard of Kubernetes by now, it means you are reading this from another planet. I didn't think I would have such a reach, awesome!

In short, Kubernetes - or its numeronym k8s - is an open-source orchestrator of containerized applications: you give it a bunch of machines (physical or virtual) and then tell it which applications you need running. It takes care of the rest: availability, management, fault tolerance, scalability, etc. Well, it is in truth a bit more complicated than that, but you get the gist.

There are so many Kubernetes tutorials out there you will find me silly proposing yet another one. However, mine is special: it completely focuses on the developer's side. What counts for a developer is to understand the basics of deploying actual apps to k8s. We are not SREs or system administrators, so we don't care how the cluster runs. But we *do* care about shipping applications that can be deployed to Kubernetes! For this, we need a basic understanding of how apps are usually deployed.

## The talk

This is exactly what I tried to explain during a presentation at the [24th Fribourg Linux Seminar: Kubernetes and Friends](https://www.meetup.com/fribourg-linux-seminar/events/292354383) given on May 11th, 2023.

Here is my talk's summary:

> *Let's embark on a journey where we cover all the essential building blocks you need to bring your applications to the cloud. We'll talk about* ***Kubernetes 101***\*,\* ***manifests***\*, the powerful\* ***Helm*** *packaging system,* ***helmfile***\*, and more. By the end of the talk, you'll have a good understanding of the different technologies to package, deploy, and manage your dockerized applications on Kubernetes with style.\*

As always, I couldn't stop myself and also created a complete step-by-step tutorial available online to accompany my talk, which I want to share with you all.

*For those of you speaking* ***⚠ French ⚠****:*

%[https://www.youtube.com/watch?v=93oW5aGz-oo] 

## The tutorial

The complete tutorial is available at ⮕ ✨✨ [derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro](http://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro) ✨✨

**A Cool Kids Guide to Deploy on Kubernetes:**

1. [Introduction](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/)
    
2. [The Cluster](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/00-sks/)
    
3. [Manifests](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/01-deploy-raw/)
    
4. [Helm](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/02-helm/)
    
5. [Helmfile](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/03-helmfile/)
    
6. [Argo CD](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/04-argo/)
    
7. [Summary](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/05-summary/)
    

**IMPORTANT**: I use the [Scalable Kubernetes Service (SKS)](https://community.exoscale.com/documentation/sks) from [Exoscale](https://exoscale.com/) in the tutorial, but if you don't want to spend money on this, you can follow along with a local k3d cluster. Simply follow the instructions [at the bottom of "The Cluster" section](https://derlin.github.io/fribourg-linux-seminar-k8s-deploy-like-a-pro/00-sks/#want-to-work-locally).

---

As always, if you enjoy this, please like the article and leave a :star: on the Github repo ⮕ [https://github.com/derlin/fribourg-linux-seminar-k8s-deploy-like-a-pro](https://github.com/derlin/fribourg-linux-seminar-k8s-deploy-like-a-pro)!

With love, derlin.