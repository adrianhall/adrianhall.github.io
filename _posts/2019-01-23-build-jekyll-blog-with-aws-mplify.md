---
title: "Build and publish a Jekyll powered blog easily with AWS Amplify"
categories:
  - Cloud
tags:
  - Jekyll
---

I have a couple of blogs. Firstly, I consolidate all the blogs that I write centrally on [my own github.io](https://adrianhall.github.io) site, and secondly, I run [a weekly link consolidation blog](https://adrianhall.gitlab.io/awsmobile-weekly/) that lets developers like you know what is going on in the world of AWS Amplify. Both of these are currently written using the Jekyll platform using the hosting of their respective source code control platforms, but I want to move them. Firstly, the Github and Gitlab hosting facilities have a limited set of plugins, whereas Jekyll supports many more plugins (and I want to use some of them), and secondly, I’d like to move them onto their own domains, decoupling them from where the blogs are stored.

When publishing your own blog, there are a couple of problems you need to solve:

* What is your workflow for writing blogs?
* How do you preview blogs, including sharing previews of blogs?
* How can you automatically publish new blogs?

AWS Amplify has a nice CI/CD platform for web applications called the AWS Amplify Console. This allows you to hook your source code control system (Github, Gitlab, Bitbucket, CodeCommit, etc.) to a continuous deployment system. In this article, we will take a look at how to leverage the AWS Amplify Console to quickly and easily enable a great workflow for publishing your blog.

## The writing process

I’m not going to tell you how to build a Jekyll site. Firstly, this has been covered ad nauseum (although [the official documentation](https://jekyllrb.com/docs/) has a good run through and I like [this blog](https://www.sitepoint.com/jekyll-sites-made-simple/) from Sitepoint). Secondly, I’m more interested in how to develop a workflow to publish blogs.

Once you have a Jekyll site, you will have a bunch of source files in a directory. Blogs for Jekyll are written in [Markdown](https://daringfireball.net/projects/markdown/). You create a text file called something like `_posts/2019–01–25-thispost.md` and fill it in with Markdown formatted text. In addition, you might want to add some Jekyll specific directives using Liquid to do internal linking (like linking to another blog, or syntax highlighting).

I use Visual Studio Code for editing blogs. [Visual Studio Code](https://code.visualstudio.com) already knows about editing markdown, but there are a couple of extensions that enable live preview and syntax highlighting. I personally like [Markdown All in One](https://marketplace.visualstudio.com/items?itemName=yzhang.markdown-all-in-one), which incorporates all the bits you are looking for.

I also check in my posts to a Git repository linked to either Github or GitLab. I specifically create a new branch on the git repository. This is normally called nextissue or the title of the blog. I do all my work in a “non-master” branch of the repository.

```bash
git checkout -b nextissue
```

So, what’s the process:

* Open the Jekyll directory in Visual Studio Code.
* Create the appropriate file in the `_posts` directory and add [Front Matter](https://jekyllrb.com/docs/front-matter/).
* Start a terminal (within VS Code) and run `bundle exec jekyll serve --future`.
* Start a web browser on my second screen that points to the local copy of my blog (usually on http://localhost:4000).
* Write my blog.

At this point, I’ve got VS Code with a vertically split screen — the left is the raw markdown and the right is a good facsimile of the content (but not everything). The split screen allows me to see live rendering of the page to ensure that numbers flow correctly, headings are correct, and so on. When I am happy with my blog content, I’ll switch over to the web browser and hit refresh. This will show me what it actually looks like to other people, including the liquid tags, syntax highlighting, and so on.

Once I’ve got the blog the way I want it, I will check it into git and push the change to my remote (Github or GitLab). My new blog is now staged in my branch.

To publish the blog, I merge that branch into the master branch:

```bash
git checkout master
git merge nextissue
```

My new blog is now in the master branch. Git has several advantages for me. I can keep blogs local by not pushing the branch to the remote. I can keep several blogs going at once easily by using separate branches. I can easily throw away blogs by just removing the branch. I can also do formatting and template changes separate from blog production.

## Publishing the production blog

Now that I’ve got a master branch with my site in it, I want to publish it. This should be a one time activity. In other words, I want the blog to be published whenever I push to master. I don’t want to have to run through a complicated process to publish a blog. It should “just happen”. This is where AWS Amplify Console comes in:

* Log onto the [AWS Amplify Console](https://console.aws.amazon.com/amplify/home?region=us-east-1#/).
* If necessary, click the **Get Started** button underneath **Deploy**.
* Select your source code repository, then click **Next**.
* Log into your source code repository.
* Authorize AWS Amplify Console to access your source code.
* Select the repository you want to publish.
* Select the branch you want to publish (in this case, it’s `master`).
* Click **Next**.
* Confirm the settings by clicking **Next**. AWS Amplify already knows about Jekyll, so no changes are required.
* Click **Save and Deploy**.

Your site will take a few minutes to be ready (depending on how long your site takes to build). Once deployed, you can click on the provided link in the console to access your site.

There are some caveats.

If you have published a blog on Github or GitLab pages, then you may have included a `baseurl` directive in the `_config.yml` file. Since the base URL of the site is the top of the site, you don’t need this directive using AWS Amplify Console. This manifests as a plain (non-styled) site. You can either remove the directive from the `_config.yml` file or adjust the build command. To adjust the build command:

* Click on **Build Settings** in the left-hand menu.
* Click **Edit**.
* Adjust the build command to be `bundle exec jekyll build --baseurl /`.
* Click **Save**.
* Go to your app (under **All apps**), then select the **Build** step.
* Click **Redeploy this version**.

The second caveat is that AWS Amplify Console tunes the site by default for faster deploys but slower performance. This is ideal when developing, but not as good for slower moving production sites. You can adjust this by tuning the CDN settings:

* Click on **General** under **App Settings**.
* Select the `master` branch from the list.
* Select **Action** > **Adjust TTL**.
* Set the TTL to your desired value. For example, I use 4 hours on my own blog (because I do correct spelling after launch), but you can use whatever value you want. On a production site, the longer the time, the longer the data is cached on the CDN.
* Click **Save**.

It takes approximately 15 minutes to take effect.

The other thing you will likely want to do is to purchase a DNS domain name and host it in [Amazon Route53](https://aws.amazon.com/route53/) — the AWS DNS service. This allows you to set up the DNS records for your blog easily. If you are hosted with a third party DNS provider (for example, DomainMonger or GoDaddy), then you have a little bit more work to do for appropriate domain registration.

## Publishing the development blog

I have two branches for my blog — a production blog and a development (next issue) blog. Sometimes, I want to share a blog that is under development with others prior to publication. It’s almost always a good idea to get a second (or third) set of eyes on stuff you publish. You will never write a completely correct blog the first time. It’s like writing bug-free code. It never happens to me.

You can connect your `nextissue` branch to the AWS Amplify Console and then password protect it. Give your reviewers a username and password.

First, let’s add a new branch for publication on the AWS Amplify Console:

* Go to your app within the AWS Amplify Console.
* Click **Connect branch**.
* Select the branch, then click **Next**.
* In the **Configure build settings** screen, adjust the build command to `bundle exec jekyll b --baseurl / --future`.
* Click **Save to my repository**. This adds an `amplify.yml` file to your branch.
* Click **Next**, then **Save and deploy**.

Note that the `amplify.yml` file is checked into source code. If you now merge the `nextissue` into `master`, your new build spec will be on production. To prevent future posts from appearing on the production build, you can use the (provided) `AWS_BRANCH` environment variable and a bit of bash shell scripting to adjust the build command based on the branch. An example build spec is as follows:

```yaml
version: 0.1
frontend:
  phases:
    preBuild:
      commands:
        - bundle install
    build:
      commands:
        - if [ "${AWS_BRANCH}" = "master" ]; then bundle exec jekyll b --baseurl /; else bundle exec jekyll b --baseurl / --future; fi
  artifacts:
    baseDirectory: _site
    files:
      - '**/*'
  cache:
    paths: []
```

Now, let’s set up some access controls:

* Click **Access control** in the left-hand menu.
* Click **Manage access**.
* Next to your branch, select **Restricted — password required**.
* Enter a username and password in the boxes provided. I use “reviewer” for the username and a complex password.
* Click **Save**.

Go back to the main app page and click the link for this branch. You will note that it uses basic authentication to request a username and password for the site. You can now provide that username and password to your reviewers.

## Wrap up

There is a lot more that the AWS Amplify Console can do for you, but this covers the basics to get a good Jekyll blog site working using your own domain and have it distributed all over the world so that your readers get quick access to the information you are providing.

Welcome to the world of blogging!

{% include links.md %}
