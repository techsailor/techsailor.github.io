---
layout: post
title:  "4 Ways to use Jekyll with Github pages"
date:   2014-08-11 20:20:51
categories: jekyll github-pages github
---
##Jekyll

[Jekyll](http://jekyllrb.com/docs/home/) is a static website
generator. It traverses a directory structure searching for content
files. The files are pumped through a pipeline of processors
generating output files. Pages can refer to master layouts and can
include other files. This provides a framework that supports a
modular, yet simple approach to building a website.

The input files can be either html or markdown. If they don't contain
the Frontmatter signature : `---` they are copied as-is, if they
contain [Fronmatter](http://jekyllrb.com/docs/frontmatter/), they are
processed.

Processing involves passing the content through filters such as
[Liquid](http://jekyllrb.com/docs/templates/) templates. This adds
amongst other things, variables, which can be used in templates. Their
values can be set in yaml data files, or as a yaml Frontmatter at the
head of the individual files.

##Github pages

Github pages have existed for sometime now, and is a handy free
hosting resource for small websites. 

Github have recently extended their documentation system, so that it
processes via Jekyll. Making github pages a very powerful publishing
service for small websites and blogs.

## Getting setup

There are 4 ways to get started with Jekyll at Github with your
personal or project web presence. 

They all centre on creating a git repository to work on, either at
Github and then you clone to get a local copy or you create one
locally. To publish you need to push your changes to a particular
branch on your Github repo, this triggers Github to process that
branch and publish your website.

###It's all in the name

Github provide 2 basic publishing mechanisms.

1. Account level (Either user or organisation accounts)
2. Project Level

###Account Level

Github refers to these also as User or Organisational pages. 

Here you create a repository with a particular name, so you can only
have one of these per account. The template for the name is
`username.github.io`. 

The complete remote Github repository URL becomes
`http://github.com/username/username.githib.io`

Where username is your Github account name.

The website URL becomes `http://username.github.io/`

The `master` branch must be updated to trigger a jekyll build.

Starting from a stand-alone local repository. (one which `origin` has
not been set up).

    $ git checkout -b master
    # make sure we are on the correct branch
    $ git remote add origin <remote repository URL>
    # Sets the new remote
    $ git remote -v
    # Verifies the new remote URL

Then

    $ git push origin master:master

Will push your local `master` branch to the `origin` `master` branch.


###Project Level

This give you a documentation area for your project, so the repository
is your project repository, but you must the `gh-pages` branch to
publish your changes.


The complete remote Github repository URL becomes
`http://github.com/username/your-project-name`

The website URL becomes `http://username.github.io/your-project-name`

The `gh-pages` branch must be updated to trigger a jekyll build.


The [Github instructions](https://pages.github.com/) can be read for
more information.

###Assumptions

Let's say I want to create a user website and my account name is named
*techsally* at Github.

## Method 1 

###Create at Github website.

> Go over to Github, Login, Create a new repository, but name it like
> so : `techsally.github.io`

Locally you can now clone your repository.

    $ git clone https://github.com/techsally/techsally.github.io
    $ cd techsally.github.io

To check where you are :

    $ git branch -v
    $ git status

Github only processes and publishes the changes to the master branch
in this repository.

    $ git checkout master # Just to be sure you are on the master branch

Create a file called index.md and give it some contents, eg. :

    ---
    layout: default
    ---
    # Welcome to my fron page

    Content is here

Add, commit and push changes to your origin master branch.

    $ git add index.md
    $ git commit -m "Added index.md"
    $ git push origin master

To view the github pages site the url would be :

    https://techsally.github.io/

## Method 2

###Using Github's *Automatic Page Generator*

####The Basic flow

Under your project settings, you use the *Automatic Page Generator*
option. Clicking this brings up a pre filled edit form for the page
(values are taken from the repository name and description and other
meta data). There's even an input field for a Google Analytics
ID. After completing this page, you continue on to a preview of your
page. You can select which template to use from a set of predefined
page templates. Publish sets up your repo at github and this can be
cloned locally in the standard fashion.

Github describes process
[here](https://help.github.com/articles/creating-pages-with-the-automatic-generator).


## Method 3

### Start on your local repository first.

I would imagine that this will become the preferred method to set-up
Jekyll and it's dependencies eventually, although presently the
describe process seems a bit flawed, only as there seems to be a boot
strap step missing which I will attempt to solve.

This does involve a little more work since you need to have a version
of `ruby` and the Ruby package manager *`Bundler`*
installed, ([see steps 1 & 2 in this Github help page](https://help.github.com/articles/using-jekyll-with-pages#installing-jekyll).

However, rather than create a Gemfile in step 3. I would after installing bundler with,

    gem install bundler

Install jekyll (and dependencies) with

    gem install github-pages

Let me explain. *github-pages* is a Gem which has been produced by
Github as a sort-of meta-gem, a wrapper around Jekyll and it's
dependencies. So when you install github-pages, you get jekyll
installed too. Using a Gem manufactured by Github must be prudent,
since one would expect their gems to be in line with what they
actually have running on their production servers.

So as a consequence, we now have a version of Jekyll locally, and I
suggest this is used to create the new skeletal project directory.

so Step 3 should be,

3. Create project directory with Jekyll

    $ Jekyll new project

What you get when Jekyll creates a project template is the following.


    ../new-jekyll/
    ├── about.md
    ├── _config.yml
    ├── css
    │   └── main.css
    ├── feed.xml
    ├── _includes
    │   ├── footer.html
    │   ├── header.html
    │   └── head.html
    ├── index.html
    ├── _layouts
    │   ├── default.html
    │   ├── page.html
    │   └── post.html
    └── _posts
        └── 2014-08-13-welcome-to-jekyll.markdown

There are plenty of examples in their stock pages, and the structure
is naturally Jekyll, which is easy to add to. Obvious this only
creates a new project directory. You will need to initialise git and
add the remote origin so that you can push your changes, as described
eariler.

If you want to upgrade at a later date,

    gem update github-pages

Will bring the gems in-line with the latest versions.

## Method 4

### Fork the Jekyll project

Forking the original Jekyll project will ultimately give you full
control and allow you to stay at the tip of the stream of updates.

Having forked at github, clone your Jekyll repository to get a local
copy and start a your website branch. This allows you to add an
upstream remote so that you can fetch upstream changes, and rebase
your work against it.

     git remote add upstream https://github.com/jekyll/jekyll.git
     git checkout website_develop
     git fetch upstream master
     git rebase upstream/master
     ...

You would have cloned your Github repository, so you should already
have an remote named *origin*. You can check this with :

    git remote -v

or

    git branch -av

To push your changes, to your specially named Github master branch for
publishing, you first need to add a reference to your remote
repository.

    git remote add website https://github/com/username/username.githib.io.git

And then publish with

{% highlight bash %}

$ git push website website_develop:master

{% endhighlight %}


Diving a little deeper into the Github deployment of Jekyll you'll
discover that Github allow only a restricted set of plugins and always
run Jekyll in [safe
mode](https://help.github.com/articles/using-jekyll-with-pages#configuration-overrides).

However, this does not prevent you from installing and generating your
site locally. Then arrange for the generated _site directory to be
published to Github pages, instead of push the source files for remote
Jekyll processing.

----

####Further reading

* Github Help Document [Using Jekyll with pages](https://help.github.com/articles/using-jekyll-with-pages)
* Jekyll Documentation [Github pages with Jekyll](http://jekyllrb.com/docs/github-pages/)