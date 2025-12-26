---
title: My Hugo + Papermod, Decap CMS, Umami, and Cloudflare setup
date: 2025-12-16T20:20:00+11:00
draft: false
tags:
  - how-to
categories:
  - Technical
description: A documentation of all of how I set up my personal blog, complete
  with all of the bells and whistles
---
## Why not Ghost?

Alright here's the scenario: I have almost nothing to do at the moment. I've finished my final exams, don't currently have a job, but want / need some sort of creative outlet.

And then it comes to me - a blog! They don't seem too hard to set up; I've seen one of my friends set up Ghost before, and all it took was a docker compose, right?

I was then shot 57 times.

My problems with Ghost are such:

1. SMTP is **required**. I'm planning on setting up a newsletter at some point (not like anyone actually cares but yk, it'd be cool), but I don't want to have to go through all of that setup and troubleshooting (I'm looking at you, smtp.mailgun.com vs smtp.mailgun.org) before I even get to make a single post. It destroys my inspiration
2. I personally really want to self-host as much as possible. While yes, Ghost supports it, there are some major issues that I have, the first being that my router is behind CGNAT, which means Cloudflare tunnels and needing to set up nginx configs. Again, while entirely possible, I am lazy. 
3. The other disadvantage to self hosting is that while Ghost isn't heavy, it's certainly not light. That, in addition to ideally a self-hosted tinybird instance (which for my initial run I didn't actually do - I just used the official hosting) just made it a little too much for me to stomach with my little Pentium G4400 and 8GB of DDR3 RAM. 
4. That, and even if I was running on absolute monster of a server, I still live in Australia, and that means *at least* [200ms](https://wondernetwork.com/pings/Paris/Sydney) of latency for every single request to Europe, and more for the US. As someone who likes my websites responsive and uninterrupted as possible (*cough cough* cookie banners)

So I decided to pivot, and after a little bit of googling, the name that kept popping up over and over again was Hugo, the customisable static site generator that promised to be the end to all of my problems.

## The actual setup

I'm going to split this up into different sections so that you don't actually have use all of them, because I mean, it's not like they actually interact. The only one that's really necessary is Hugo + PaperMod for the specific configurations, but most themes have their own ways to add headers and partials, which is really all that you need.

### Hugo + PaperMod

For a local instance of Hugo (I'm going to omit the "+ PaperMod" now), you'll first need to install it. Personally, I use NixOS, and it was as simple as adding:

```lua
environment.systemPackages = with pkgs; [\
  hugo\
];
```

To anywhere within my system config (I might make a post about this later)

For other systems, you can find how to install Hugo on their [website](https://gohugo.io/installation/).

Next, it's as simple as 

```shell
hugo new site MyNewBlog --format yaml
cd MyNewBlog
git init
git submodule add --depth=1 https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
git submodule update --init --recursive # needed when you reclone your repo (submodules may not get cloned automatically)
git submodule update --remote --merge # For if you want to update the theme
```

Then, in your text editor of choice, open up the `hugo.yaml` file and add the following:

```yaml
baseURL: https://example.org/
languageCode: en-us
title: My New Hugo Site
theme: [ "PaperMod" ] # Add this
```

From the `MyNewBlog` directory, you should now be able to run `hugo serve -D`. If you then navigate to <http://localhost:1313>, you should now see something that looks like this:

![](/uploads/screen-shot-2025-12-27-at-07.11.47.png "Your new Hugo blog")

> **Note:** You should keep `hugo serve -D` running throughout the tutorial. It will update to your changes automatically, and lets you see whatever you're working on
> Now, clearly from here you'll have a bit more configuring to do. So let's start with some of the easier configuration.

```md
---
title: First Post
---
# Hello, world
This is my first blog post!
```

Now, to be fair, this isn't quite the kind of site that you were probably looking for. There are A LOT of config options in both Hugo and PaperMod. You can find most of the configuration options on the [PaperMod Wiki](https://github.com/adityatelange/hugo-PaperMod/wiki/Features), and the list of all of Hugo's native config options on the [Hugo Wiki](https://gohugo.io/configuration/all/) (if I'm being honest, I haven't really looked at all of the Hugo native options just yet).
Here's *most* current config. You'll find other add-ons later, including a full config at the very end:

```yaml
baseURL: https://examplesite.com/ # Replace
languageCode: en-us
title: MyHugoBlog # Replace
theme: ["PaperMod"]

enableRobotsTXT: true
buildDrafts: false
buildFuture: false
buildExpired: false

minify:
  disableXML: true
  minifyOutput: true

outputs:
  home:
    - HTML
    - RSS
    - JSON

params:
  env: production
  title: My Hugo Blog # Replace
  description: "An example description" # Replace
  keywords: [Blog, Portfolio, PaperMod]
  author: quexeky
  images: ["/apple-touch-icon.png"]
  DateFormat: "January 2, 2006"
  defaultTheme: auto # dark, light
  disableThemeToggle: false

  ShowReadingTime: true
  ShowShareButtons: true
  ShowPostNavLinks: true
  ShowBreadCrumbs: true
  ShowCodeCopyButtons: true
  ShowWordCount: false
  ShowRssButtonInSectionTermList: true
  UseHugoToc: true
  disableSpecial1stPost: false
  disableScrollToTop: false
  comments: true
  hidemeta: false
  hideSummary: true
  showtoc: true
  tocopen: false

  assets:
    disableHLJS: true # to disable highlight.js
    disableFingerprinting: false # Ensures that cache is updated whenever new content is added

  label:
    text: "Home"
    icon: /apple-touch-icon.png
    iconHeight: 35

  # home-info mode
  homeInfoParams:
    Title: "Hi there \U0001F44B"
    Content: | 
      This is an example Content description # Replace

  socialIcons:
    - name: github
      url: "https://github.com/YOUR_GITHUB_USER" # Replace
    - name: linkedin
      url: "https://linkedin.com/in/YOUR_LINKEDIN_USER" # Replace

  cover:
    hidden: true # hide everywhere but not in structured data
    hiddenInList: true # hide on list pages and home
    hiddenInSingle: true # hide on single page


menu:
  main:
    - identifier: tags
      name: tags
      url: /tags/
      weight: 20
    - identifier: search
      name: search
      url: /search/
      weight: 1
    - identifier: rss
      name: RSS
      url: /index.xml
      weight: 40

# Read: https://github.com/adityatelange/hugo-PaperMod/wiki/FAQs#using-hugos-syntax-highlighter-chroma
pygmentsUseClasses: true
markup:
  highlight:
    noClasses: false
    anchorLineNos: false
    codeFences: true
    guessSyntax: true
    style: monokai
    lineNos: true
```

#### Setting up a GitHub Repo

The final thing here that is required for a few of the later options is setting up a [GitHub](https://github.com) repo. I'm going to assume that you already have a working GitHub account with a local \[Personal Access Token](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#using-a-personal-access-token-on-the-command-line).

All you'll need for this is to go to GitHub, and click the green `New` button in the top left.

![](/uploads/screenshot_20251227_085431.png)

From there, ensure that it's an empty repository. Set it to public if you want to enable \[Comments with Giscus](https://blog.quexeky.dev/posts/#comments-with-giscus), set it to public. Otherwise it doesn't matter.

![](/uploads/screenshot_20251227_085547.png "An example empty GitHub repository")

From here it's as simple as (from your `MyNewBlog` directory):

```shell
git init .
git add .
git commit -m "Initial commit"
git remote add origin http://github.com/YOUR_GITHUB_USERNAME/YOUR_GITHUB_REPO_NAME.git
git push --set-upstream origin master # See Note

# From here, if you want to update the remote, you can just run:
git push
```

> **Note**: If you're prompted to provide a username and password here, see the [GitHub Docs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#using-a-personal-access-token-on-the-command-line)

### Search with FuseJS

One of the easiest things to add to your new blog is a [search page](/search). It requires only a few more lines of configuration, and makes the experience so much better for people looking for specific posts.

The first thing you'll want to do is create a `search.md` page under the `contents` directory like so:

```
MyNewBlog
└── content
    ├── posts
    │   └── first_post.md
    └── search.md
```

And fill it with the following:

```yaml
---
title: "Search" # in any language you want
layout: "search" # necessary to enable search
url: "/search"
# description: "Description for Search"
summary: "search"
placeholder: "Hugo, Cloudflare" # The placeholder text that will appear in the search bar
---
```

Then in your `hugo.yaml`, you can add the following under the `params` field (it works without it because PaperMod includes it by default, but I find that these settings work well):

```yaml
params:
  # for search
  # https://fusejs.io/api/options.html
  fuseOpts:
    isCaseSensitive: false
    shouldSort: true
    location: 0
    distance: 1000
    threshold: 0.4
    minMatchCharLength: 0
    ignoreLocation: true
    limit: 10 # refer: https://www.fusejs.io/api/methods.html#search
    keys: ["title", "permalink", "summary", "content", "tags"]
```

Now if you navigate to <https://localhost:1313/search>, you should be able to search your existing posts

> **Note:** If you start seeing duplicate posts, you might have edited a post while Hugo cached it (this is the fingerprinting in the config file). You'll just have to run `hugo --cleanDestinationDir` to get rid of any duplicates. This only affects local builds though



### Comments with Giscus

This one here is ever so slightly more in the weeds. As a reminder though, you're always welcome to skip over any of these sections and come back to them later. The only one that really matters is having the actual Hugo site running.

With that said, comments are fairly interesting, because they allow your audience to interact with you. I personally use [Giscus](https://giscus.app/), although there are many other alternatives out there, depending on your specific needs. 

For some context, Giscus relies on the GitHub comments API to store comments, meaning that users need to sign in with their GitHub accounts to leave comments. I personally like this, because my target audience is people who are likely to have a GitHub account already, and this way I can prevent spam. 

Regarding the actual setup, PaperMod provides us with a very easy API to enable comments. First though, we're going to need a bit of setup through the [Giscus Configuration](https://giscus.app/):

1. Ensure that your repository is set to public. This should have been done in \[Setting up a GitHub Repo](https://blog.quexeky.dev/posts/my-hugo-papermod-umami-and-cloudflare-setup-2025-12-16/#setting-up-a-github-repo)
2. Install the [Giscus app](https://github.com/apps/giscus) by clicking \`Configure\` and then selecting your user, and then choosing your GitHub repo

![](/uploads/screenshot_20251227_091617.png)

3. In the `Settings` page for your repo, navigate to `General > Features` and then `Set up Discussions`. GitHub will prompt you to create a new discussion, which you can delete later. I personally also go through and delete all of the 

![](/uploads/screenshot_20251227_092257.png)

4. I personally also go through and delete all of the other discussion categories and pressing the pen icon next to \`Categories\`, and rename my announcements channel to something like "Comments," so that it looks like this at the end

![](/uploads/screenshot_20251227_093646.png)

4. Navigate back to [Giscus](<>), and double check that when you put `YOUR_GITHUB_USERNAME/YOUR_BLOG_REPO_NAME` into the `repository:` field, it says "Success! This repository meets all of the above criteria."
5. Configure settings. Here are my personal settings:

![](/uploads/screenshot_20251227_092823.png)

Almost finished. Once you're finished configuring everything, you'll be able to find a html script tag under "Enable Giscus" that looks like this:
```html
<script src="https://giscus.app/client.js"
        data-repo="quexeky/MyNewBlog"
        data-repo-id="R_kgDOQvZchg"
        data-category="Announcements"
        data-category-id="DIC_kwDOQvZchs4C0RPG"
        data-mapping="pathname"
        data-strict="1"
        data-reactions-enabled="1"
        data-emit-metadata="0"
        data-input-position="top"
        data-theme="preferred_color_scheme"
        data-lang="en"
        data-loading="lazy"
        crossorigin="anonymous"
        async>
</script>
```
Now you'll want to create a partials folder under layouts, and then a comments.html file under that:
```

MyNewBlog
├── content
│   └── posts
│       └── first_post.md
└── layouts
    └── partials
        └── comments.html
```
Put your entire script tag into the comments.html file, and if you navigate to <http://localhost:1313/posts/first_post.md>, you should see your comments showing up at the bottom of your post!
