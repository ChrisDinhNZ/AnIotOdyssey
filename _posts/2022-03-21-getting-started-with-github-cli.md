---
layout: post
title: Getting Started With GitHub CLI
date: 2022-03-21 23:18 +1300
background: '/assets/posts/2022-03-21-getting-started-with-github-cli/post-banner-2022-03-21-getting-started-with-github-cli.jpg'
Tag:
    - GitHub CLI
    - Git
    - Tooling
---

One of the most powerful tool in our toolbox as a developer is the ability to [version control](https://en.wikipedia.org/wiki/Version_control) our work. This enables us to work in an agile environment where we can make changes with more confidence.

I personally version controls a lot of my works. These works are not just limited to source codes but anything that can change over time e.g. blog contents, ideas and designs, proof of concept projects.

There are many `version control platforms` out there ([GitHub](https://github.com/github), [GitLab](https://about.gitlab.com/), [BitBucket](https://bitbucket.org/)).

For myself, I am using `GitHub`. My workflow generally involves creating a remote repository on GitHub (up till now this has been done via a browser), then create a local repository on my development machine by cloning the remote repository (by the way, if you are not familiar with some of these terminologies, have a read at this [About Git](https://docs.github.com/en/get-started/using-git/about-git)). Yes there is a bit of context switching involved so I thought this would be a good opportunity for us to have a look at the [GitHub CLI](https://github.com/cli/cli) tool.

## Installing GitHub CLI

There are a few ways we can [install GitHub CLI](https://github.com/cli/cli#installation) which also depends on our development environment. I am currently using Windows so the easiest way is to download the installation package from the releases page under the `assets` section. As of writing, the latest release is [GitHub CLI v2.6.0](https://github.com/cli/cli/releases/tag/v2.6.0)) and for a `Windows x64` system, we can download [gh_2.6.0_windows_amd64.msi](https://github.com/cli/cli/releases/download/v2.6.0/gh_2.6.0_windows_amd64.msi).

Once installed, we can verify the installation by running the command `gh --version`.

![Check GitHub CLI version](/assets/posts/2022-03-21-getting-started-with-github-cli/check_github_cli_version.gif)


## Authenticating GitHub CLI

Next, we will need to authenticate with our GitHub account by running the command `gh auth login`. The people at GitHub has made the process really simple so just follow the instructions. Note in the example below we used a [personal access token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token) instead of password to authenticate.

![GitHub CLI login](/assets/posts/2022-03-21-getting-started-with-github-cli/github_cli_login.gif)

## Creating a new repository

There's two methods of creating a new GitHub repository, an interactive method and a non-interactive method.

To use the interactive method, run the command `gh repo create`. We will then be guided through the options needed for the new repository.

To use the non-interactive method, run the command `gh repo create` but this time explicitly providing the options with the create command. For example: `gh repo create GitHubCLIDemo -d "Create command demo" -g VisualStudio -l MIT --public -c`. For more help on the available options, have a look at the the [gh repo create](https://cli.github.com/manual/gh_repo_create) page.

![Create new GitHub repository](/assets/posts/2022-03-21-getting-started-with-github-cli/github_cli_create_repo.gif)

As you can see, it is very easy to create a new repository on GitHub and clone it to our local environment without the need to switch to a browser. For me, this workflow alone makes GitHub CLI an awesome addition to our development toolkit.
