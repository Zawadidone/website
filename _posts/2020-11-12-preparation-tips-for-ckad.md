---
title: Preparation Tips for CKAD
date: 2020-11-12
header: 
  teaser: "/assets/img/kubernetes.png"
---

5 days ago, I took the CKAD exam and passed. So I decided to write this short blog post about my experience and to share some tips that helped me pass it. Everything in this blog post is about my experience, you may be able to pass the exam easily without these tips. Only this has worked best for me.

### Background
I have been working with Kubernetes for more almost a year. One and a half year ago I did my internship about automating a continuous integration and continuous delivery (CI/CD) pipeline using Gitlab, Docker and Kubernetes. From that point, I started using Kubernetes for work and for some hobby projects. To learn more about Kubernetes I read a lot of documentation and often had to debug applications to make them "Kubernetes" proof to deploy to production. 
During my work or for hobby projects I almost never use `kubectl`  only to debug pods that are not working or testing manually stuff. Most of the time I use Gitlab CI/CD that upgrades a Helm chart with the command `helm upgrade -i [...]`.   A week ago I saw this [webinar](https://youtu.be/z0VSJPdP674) about the CKA/CKAD exams I thought why not do it this weekend. **NOTE: before register for the exam just search on Google for vouchers of the Linux Foundation it can save you a few bucks.**

### Tips

The exam is all about hands-on experience using `kubectl`, you don't have set up clusters this exam is all about development. During the exam, you're allowed to have **one** tab open in your browser with the Kubernetes documentation. This is handy because you cannot know everything out of your head and if you forget something you can find it.  To easily perform the exam, experience with the following software packages is required.
 
* Linux (basics understanding)
* BASH basic commands `cat, cp, mv` 
* SSH
* Kubectl

The exam consists of 19 questions you have to finish in 2 hours so time is your enemy. To quickly edit yaml files and using `kubectl` I added the following VIM config to `~/.vimrc` and enabled auto-completion for `kubectl` (No more typo's).

```bash
vim ~/.vimrc
imap jj <Esc>
syntax on
set ts=2
set sts=2
set sw=2
set expandtab
set nu
# enable kubectl auto completion https://kubernetes.io/docs/tasks/tools/install-kubectl/#enable-kubectl-autocompletion
```

Because I already have experience using `kubectl` I studied on Friday and Saturday evenings and took the exam on Sunday afternoon. To prepare for the exam questions I used the following resources:

* [https://github.com/dgkanatsios/CKAD-exercises](https://github.com/dgkanatsios/CKAD-exercises)
* [https://github.com/twajr/ckad-prep-notes](https://github.com/twajr/ckad-prep-notes)
* [https://medium.com/bb-tutorials-and-thoughts/practice-enough-with-these-questions-for-the-ckad-exam-2f42d1228552](https://medium.com/bb-tutorials-and-thoughts/practice-enough-with-these-questions-for-the-ckad-exam-2f42d1228552)

Before applying a yaml definition always use the parameters `kubectl run [...] --dry-run=client -o yaml > <file.yaml>`. This allows you to edit the yaml before creating the object. My routine consisted of the following:

1. Run can also be replaced with `create` but this depends on the object you want to create. 
		`kubectl run <name> --image <name> <paramaters> --dry-run=client -o yaml > <file>.yaml` 
2. Edit the object based on the question.
		`vim <file>.yaml`
3. Check if it works 
		`kubectl create -f <file>.yaml --dry-run=client`
4. `kubectl create -f <file>.yaml` 
5.  Check if resource is created succesfully. 
			`kubectl logs <pod-name>`, `kubectl get <pod/deployment/job/cronjob> <pod-name`,  `kubectl describe <pod/deployment/job/cronjob> <name>` 

### Different Kubernetes versions
After finishing the exam I almost thought I made a mistake. Almost every resource you find online about this exam uses the following command to create a pod.
`kubectl run <name> --image <image> --restart Never`. This is because in earlier versions of Kubernetes (< 1.18) it was possible to create for example deployments with the command `kubectl run` ([https://github.com/kubernetes/kubernetes/pull/87077](https://github.com/kubernetes/kubernetes/pull/87077), [https://medium.com/@alexellisuk/kubernetes-1-18-broke-kubectl-run-heres-what-to-do-about-it-2a88e5fb389a](https://medium.com/@alexellisuk/kubernetes-1-18-broke-kubectl-run-heres-what-to-do-about-it-2a88e5fb389a)) so the parameter `--restart Never` was required to create just a pod. Because the exam used Kubernetes version 1.19 this luckily wasn't needed to create a pod and the command `kubectl run <name> --image <image>` was just enough without `--restart Never`. **NOTE: always check the version number noted in the [curriclum](https://github.com/cncf/curriculum), also use this version when training for the exam**


After finishing the exam on Sunday 2 days later I received my certification with a score of 91%.


<div data-iframe-width="270" data-iframe-height="270" data-share-badge-id="8872d40c-fb25-4076-9cdc-210707bba6b0" data-share-badge-host="https://www.youracclaim.com"></div><script type="text/javascript" async src="//cdn.youracclaim.com/assets/utilities/embed.js"></script>
