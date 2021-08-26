---
title: This is my first post.
description: This is a post on Test Blog about setting up a static site.
date: 2021-08-25
tags:
  - eleventy
layout: layouts/post.njk
image: https://cdn.pixabay.com/photo/2020/08/30/20/54/rice-field-5530707_1280.jpg
---

## Testing Eleventy

I have been wanting to test some static website generators for some time. I know of Hugo from a few years ago. Looks like there are a few more tools since. After reading [Jekyll vs Hugo vs Gatsby vs Next vs Zola vs Eleventy](https://mtm.dev/static), I decided to give [Eleventy](https://www.11ty.dev/) a try.

I researched a bit about easy and simple Eleventy templates to begin with and found [eleventy-high-performance-blog](https://github.com/google/eleventy-high-performance-blog) from google. Clicking the ['Use this template'](https://github.com/google/eleventy-high-performance-blog/generate) button generated a new GitHub repository of my own: [test-blog](https://github.com/teawithtanya/test-blog).

![An image](https://cdn.pixabay.com/photo/2020/08/30/20/54/rice-field-5530707_1280.jpg)

## Azure Static Web Apps

Eleventy Documentation has a bunch of [tutorials](https://www.11ty.dev/docs/tutorials/) to get started. [Deploying an 11ty Site to Azure Static Web Apps](https://squalr.us/2021/05/deploying-an-11ty-site-to-azure-static-web-apps/) (by Chad Schulz) claimed Azure had a free offering for small hobby static sites. It's [true](https://azure.microsoft.com/en-us/pricing/details/app-service/static/)!!!. And it includes a free SSL certificate for my custom domain. Sold!

## Setup

From the Azure Portal [Create Static Web App](https://portal.azure.com/#create/Microsoft.StaticApp), I configured:

*Project Details*
Subscription (Pay-As-You-Go)
Resource Group (testblog)

*Static Web App Details*
Name (test-blog)

*Hosting plan*
Plan type (Free)

*Azure Functions and staging details*
Region (West US 2)

*Deployment details*
Source (GitHub)
Organization (teawithtanya)
Repository (test-blog)
Branch (main)

*Build Details*
Build Presets (Custom)
App location (/)
Api location ()
Output location (_site)

The production GitHub Action started automatically and created a generated site at [nice-desert-09c4d691e.azurestaticapps.net](https://nice-desert-09c4d691e.azurestaticapps.net/).

Next, I used Custom domains to add a CNAME DNS record pointing test-blog to nice-desert-09c4d691e.azurestaticapps.net.