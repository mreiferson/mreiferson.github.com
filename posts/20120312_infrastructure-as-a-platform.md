# Infrastructure as a Platform

### Introduction

At bitly, infrastructure has two core responsibilities.

  1. **The obvious** - systems architecture, performance, scaling, technology choices, and implementation details.
  2. **The not-so-obvious** - new hire on-boarding, developer tools, and idioms that enable non-infrastructure engineers (science/research, application developers, etc.) to build scalable, performant, operationally sound, and easy-to-develop pieces of your product.

I say not-so-obvious because many of these things aren't pressing issues when teams are small, everyone is in one room, and it's crystal clear what the most important goal is from an engineering perspective.

As an engineering team grows, it becomes more and more important to develop and evangelize a common way of solving problems and getting things done.  Without these guidelines you end up with inconsistent solutions that have a dramatic impact on the ability to operationally handle production systems (let alone the time cost of engineers constantly re-inventing the wheel).

Maybe you're responsible 24/7 for production systems - consider any combination of the following situations:

  * logs are in 14 different places
  * an engineer chose a different one of the far too many Python web frameworks
  * services are started by the current flavor-of-the-week process manager
  * response formats from APIs differ
  * techniques used to process data vary
  * monitoring is done differently or not at all
  * backups aren't consistent
  * no tests

Sounds like one helluva mess to debug.  (and it's **always** at *3am*)

Let's talk about how we've made progress over the past year solving some of these problems.

### Development Workflow

A big win here was our switch to Git (and GitHub).  The freedom and flexibility this gave us to design a powerful code-flow (`develop` -> `review` -> `merge` -> `deploy`) has proved *extremely* valuable.

We keep our code in one repository (application code, infrastructure tools, configuration and system dependencies).  When possible we work to move things into external [open source projects](http://github.com/bitly) that are paired with an install script in our main repo.  Every engineer develops in their own fork.  Code makes it into `master` in one of two ways:

  1. **cherry-picking** - a small, well defined changeset or high-priority bug fix can be cherry-picked into `master`.
  2. **pull request** - anything else follows the process of creating a branch off `master`, developing the feature, and opening a pull request.

It should be noted that another member of the engineering team will review either of the above and provide constructive feedback, ultimately being the one to bring the code into master.  Code review is a first class member of our development workflow - it is hugely beneficial to both the developer and the reviewer.  The better you are at reading code, the better you are at writing it.  For efficiency, and to respect each others time, we often exchange commit hashes and play tit-for-tat with pull-request reviewing.

Another benefit of this model is that `master` maintains a clean history, always moving forward.  In doing so we avoid the pitfalls of coordinating a destructive history change with so many outstanding branches.  Still, we understand the value of Git's power - engineers are encouraged to re-write history to their hearts content while developing in their own branch.  When ready, a comment on the pull request stating "ready for review **@github_user**" will notify the corresponding person to begin review at their discretion.

Reviewing takes the form of comments on the pull request.  As each round of review is completed, the developer will comment "resolved" where appropriate and push additional commits to the branch.  This culminates in one final rebase and squash before a merge.  We find that looking back you rarely need the back-and-forth of many small commits and we prefer to see the change as a whole as it relates to the issue documented in the pull request.

### Developing in a Virtual Machine

If you're **not**:

  * writing code in an environment that mimics production
  * setup by the same scripts and processes that stand up production hosts
  * exposed to the challenges of making things play nice together

As soon as your code is actually running in production it's probably not going to go well.

We believe whole-heartedly in the VM approach.  Every developer (and even designers!) have a local VM for developing changes to any component in our infrastructure, front to back.

Most importantly, it forces you to design your applications to be "environment aware".  Your application code should only know the __*how*__.  The __*where*__ and __*how many*__ is the responsibility of configuration files that pivot on the *environment* the code is running in.  (see [@jehiah](http://twitter.com/jehiah)'s [post](http://jehiah.cz/a/application-settings) for more info about how we structure this)

The benefits of this are easy to understand - if you can get it working, tested, and deployable in the VM then getting it running successfully in production should be as simple as a deploy.

However, the choice of using a VM doesn't come without challenges:

  * When you're service-oriented, as the overall architecture grows, it becomes increasingly difficult to "fit" *all* services in a single VM on your local machine (RAM, etc.).
  * Not *all* services are equal and different engineers tend to touch different components more often.
  * Setup, maintenance, and learning curve are a time sink.

We've only begun to scratch the surface in addressing these issues.   We've begun to logically group services so that they can be spun up/down on demand, giving some control over whats running simultaneously.  We'll surely post a follow up with our findings as we continue to make progress in this area.

### The Way of the Bitly

The value of choosing sane and repeatable solutions to common problems becomes clearer and clearer as things grow - whether it's data, engineering team size, or number of services in your infrastructure.

Often the # of operations engineers doesn't scale linearly with the # of services they have to monitor in production.  It becomes extremely important to make their lives easy by developing good frameworks and solutions for solving problems, and make those solutions readily available, well documented, and easy to use.

We begin this education process early on with new engineers.

An engineer's first day sets the tone for his future at your company.  A new engineer at bitly can expect that on the first day they'll have some shiny new hardware to play with as well as one simple goal, get your bio on the [bitly about page](http://bitly.com/pages/about).  Since we put a strong focus on working in a VM, that implies that the existing operations and infrastructure team has done their job to have all the appropriate accounts setup such as access to email, wiki, GitHub, and of course, a VM.

This simple task facilitates the process of familiarizing themselves with the dev workflow described above as well as providing the satisfaction of seeing code go into production on day 1.

We try to expose new engineers to all different facets of bitly infrastructure and encourage them to explore the codebase.  Fixing bugs, adding small pieces of functionality, and developing brand new services are common first week tasks.

We think it's important to step away from the computer, too.  We schedule internal tech talks where we discuss the successes (and challenges) of the current state of the infrastructure.  It's an open forum where questions are answered, odd names get definitions, and bitly idioms are discussed.

### EOL

There's lots more to talk about:

  * how we deploy...
  * what are some of those bitly idioms...
  * how we make technology decisions...
  * meetings and communication...

and more.  We'll save all that for future posts.

As always, if any of this sounds interesting to you, [bitly is hiring](https://bitly.com/jobs).

<div class="postmeta">
by <a href="http://twitter.com/imsnakes">snakes</a>
</div>