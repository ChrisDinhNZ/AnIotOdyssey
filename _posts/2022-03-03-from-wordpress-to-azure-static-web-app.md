---
layout: post
title: From Wordpress To Azure Static Web App
date: 2022-03-03 22:47 +1300
background: '/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/post-banner-2022-03-03-from-wordpress-to-azure-static-web-app.jpg'
Tag:
    - Blog Site Hosting
    - Tooling
    - Azure Static Webpages
    - Jekyll
---

By day I am an embedded software developer. I worked mostly in two-way digital radio communications systems such as [DMR](https://en.wikipedia.org/wiki/Digital_mobile_radio), [P25](https://en.wikipedia.org/wiki/Project_25).

In my spare time I also tinker with other technologies outside of the embedded software domain. This tinkering and the curiosity to learn has been and continue to be a rewarding journey, it has taken me down many rabbit holes but equally enlightened by them.

About a year ago, I wanted to record these experiences and the knowledge that I have accumulated so I started blogging. As an embedded software person, creating and maintaining a blog site was another learning curve. At the time I looked at [WordPress](https://wordpress.org/) and even that felt fairly involved. I was more interested in content creation than maintaining a CMS site and so I outsourced that responsibility by going with [WordPress.com](https://wordpress.com/).

While `WordPress.com` is a great platform, in my case (in term of flexibility and value for money in the long run) it did not seem like the best option. So I started looking at alternatives.

In this blog post, we will go through the process of getting a blog site up and running using `Jekyll`.

## Jekyll - a static site generator

During my search for an alternatives I came across the concept of `static site generator` (SSG). Here is a good explanation of [what is a static site generator](https://www.cloudflare.com/learning/performance/static-site-generator/). There are many [SSG](https://jamstack.org/generators/) options out there, but I chose to go with [Jekyll](https://jekyllrb.com/) for how easy it was for me to get things up and running.

**Installing Jekyll**

The first step is to install `Jekyll` on our development machine. You can follow the installation instruction based on your machine's operating system [here](https://jekyllrb.com/docs/installation).

Check that the installation was successful by verifying the installed version for the following:

* ruby
* gem
* jekyll

![Verify Ruby and Jekyll is installed](/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/demo_verify_ruby_install.gif)

**Create a new Jekyll site**

Creating a new site is as simple as running `jekyll` command with the option `new` and passing in a name for the site.

![Create new Jekyll site!](/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/demo_new_jekyll_site.gif)

We can preview the newly created site by executing `bundle exec jekyll serve`

![Preview new Jekyll site!](/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/demo_preview_new_jekyll_site.gif)

Jekyll uses themes to change the look and feel of a site. There are plenty of themes out there, some are paid while some are free. For my blog site I liked the look and feel of `startbootstrap-clean-blog-jekyll`. You can find more details on how to install the template on their [GitHub page](https://github.com/StartBootstrap/startbootstrap-clean-blog-jekyll)

**Adding and Editing Site Content**

Blog posts goes into the `_posts` folder. The file can be added manually, but we can also use the `jekyll-compose` plugin as well. To add the plugin:

* Add `gem 'jekyll-compose', group: [:jekyll_plugins]` to the Gemfile.
* In a console terminal run `bundle` again to install the gem above.

Now all we need to do is run `bundle exec jekyll post "Hello World"` to create a new post named `hello-world.md`.

![Create new post!](/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/demo_new_blog_file.gif)

We can also create stand-alone pages as well by running `bundle exec jekyll page "Misc"`.

![Create new page!](/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/demo_new_page_file.gif)

## Deploying To Azure Static Web App Using GitHub Actions

In this section we will be using [GitHub Actions](https://docs.github.com/en/actions) to deploy the site to production using a [CI/CD](https://www.atlassian.com/continuous-delivery/principles/continuous-integration-vs-delivery-vs-deployment) workflow.

The blog site's source code will be hosted on `GitHub`. At this point let's assumed everything has been commited and pushed to a GitHub repository. If you are not familiar working with GitHub, check out their documentation for creating a new repository [here](https://docs.github.com/en/get-started/quickstart/create-a-repo).

In order to get the blog site out there into the wild, it needs to be published to a server somewhere. For this we will create a `static web app` resource via the `Azure portal` to host the site. The documentation on how to create a static web app can be found [here](https://docs.microsoft.com/en-us/azure/static-web-apps/publish-jekyll#create-the-application).

## Link A Custom Domain To Azure Static Web App

By default, when we create a static web app resource, an auto-generated domain name is associated with the site. It will be something like `https://some-generated-domain.azurestaticapps.net`. However, we would probably prefer the site to be associated with something like `myblogsite.com` or `www.myblogsite.com`. These are custom domain names which we can easily map to the generated domain name. The documentation on how to map custom domains to a static web app can be found [here](https://docs.microsoft.com/en-us/azure/static-web-apps/custom-domain).

## A CI/CD Demo

Let's create a new post and push it to GitHub.

![CI/CD Demo](/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/ci_cd_demo.gif)

When a new `commit` is pushed to GitHub, a build pipeline is triggered as defined by the GitHub action, and once the build is done we should see our new post published to the blog site.

![Post CI/CD Demo](/assets/posts/2022-03-03-from-wordpress-to-azure-static-web-app/post_ci_cd_demo.gif)

That is it. We now have a build and deploy pipeline where everytime we push some changes to GitHub, the site is rebuilt and published to the Azure Static Web App.