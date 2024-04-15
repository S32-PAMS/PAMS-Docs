---
{"dg-publish":true,"permalink":"/readme/","tags":["home","gardenEntry"],"noteIcon":""}
---

# PAMS

> [!abstract] Personnel and Artefact Monitoring System (PAMS)
> **Personnel and Artefact Monitoring System (PAMS)** is an *anti-theft solution* that utilises reliable, robust technology for *precise, real-time localisation* of personnel and sensitive objects within specified environments.
> 
> PAMS is *designed* to provide *security* and enhance operational *efficiency*. It is both *scalable* and *cost-efficient*. It also integrates with existing networking infrastructures to provide *detailed analytics*, enabling effective resource monitoring and management across various sectors.

Find out more about our design

- [[Architecture\|Architecture]]
- [[Security\|Security]]

Let PAMS resolve your asset tracking needs.

## User Manuals

**Get started** here.

- [[Guides/For Building PAMS\|For Building PAMS]]
- [[Guides/For security personnel\|For Security Personnel]]
- [[Guides/For Running Hardware\|For Running Hardware]]
- [[Guides/For Running Server\|For Running Server]]
- [[Guides/For Testers\|For Testers]]

---

# How to control this manual

Repositories:

- [Manual Content](https://github.com/S32-PAMS/PAMS-ManualContent)
- [Public Hosting](https://github.com/S32-PAMS/PAMS-Docs)

Only Manual Content needs to be edited. Just merge PRs for Public Hosting repository whenever the package manager bumps versioning updates.

## Opening manual locally

Clone the [Manual Content](https://github.com/S32-PAMS/PAMS-ManualContent) repository. You can choose any markdown editor to open their files, but for full linking functionality like you see in the manual website, download [Obsidian](https://obsidian.md/) and open this repository as an Obsidian Vault, following the setup instructions.

You can now view the manual.

## Hosting manual online

Clone the [Manual Content](https://github.com/S32-PAMS/PAMS-ManualContent) repository. Download [Obsidian](https://obsidian.md/) and open this repository as an Obsidian Vault, following the setup instructions.

Once you are inside Obsidian and have opened the repository as above, go to Settings.

![Obsidian settings.png](/img/user/Attachments/Obsidian%20settings.png)

You first need to allow Community Plugins, and then Browse and Install at least the `Digital Garden` plugin. The `Git` plugin may help you if you don't wish to do git operations in a separate terminal.

Once you have the Digital Garden plugin, proceed to continue from step 5 of [this guide](https://dg-docs.ole.dev/getting-started/01-getting-started/), as steps 1 to 4 have been already done with the `capstonepams` Github account (that's why we can show you a manual as a website).

You can now maintain our manual! To stop hosting, simply go to vercel in the guide above for Digital Garden plugin and just stop hosting there.
