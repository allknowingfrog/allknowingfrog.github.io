---
layout: post
title:  "Initializing a Ruby Application with Standard"
date:   2022-01-29 15:30:00 -0600
---

After a few years of Ruby experince, I've landed on a preferred set of tools to promote consistency and general ease of development. This is a walkthrough of how I currently initialize a new project. For demonstration purposes, I'm going to work towards creating a very basic API application, called "boxspring"[^1].

I intend to produce a series of these posts, from zero to functional API application, but this first one focuses on setting up the development flow itself. The main goal will be to automatically enforce code consistency with the marvelous [Standard](https://github.com/testdouble/standard) gem. Standard is basically a more opinionated Rubocop, and that lack of configuration is its main value proposition. You may disagree with some of the conventions, but what you'll gain is the ability to pick up any other Standard-based project and know that its conventions will be _exactly the same_.

# Prerequisites

My intended audience is Ruby developers with at least some experience. I'm going to walk through initializing a `git` repository, but I'm not going to explain any of the commands in detail. Similarly, I'll be using some `rbenv` commands without providing much context[^2]. Finally, I'll assume that [Bundler](https://bundler.io/) is installed for your current Ruby version.

# Initialization

The following commands create the project directory, set the Ruby version, initialize the repo, and generate a `Gemfile`. I'm running Ruby 3.0.3 as my system default, but I prefer to explicitly anchor the version. This helps to ensure that nothing breaks in the future, even if this project is forgotten for days, months or years.

```bash
$ mkdir boxspring
$ cd boxspring
$ rbenv local 3.0.3
$ git init
$ bundle init
```

# Dependencies

These are the contents of the generated Gemfile.

```ruby
# frozen_string_literal: true

source "https://rubygems.org"

# gem "rails"
```

Let's remove the example dependency line (`gem "rails"`), and declare the Ruby version by refercing the `rbenv` file. Note that these single-quotes are intentionally inconsistent with the rest of the file. This will help prove that Standard is working in a future step.
```ruby
ruby File.read('.ruby-version').chomp
```

Next, create a `development` group and add both `standard` and `lefthook`. Lefthook is a neat little tool for managing `git` hooks. It allows you to share configuration between local copies of the repository, and ensure that everyone is running the same validations before they commit.
```ruby
group :development do
  gem "lefthook"
  gem "standard"
end
```

Lastly, install the dependencies with Bundler.
```bash
$ bundle install
```

# Configuration

Lefthook outputs a helpful hint when the gem is installed.
```bash
Post-install message from lefthook:
Lefthook installed! Run command in your project root directory 'lefthook install -f' to make installation completed.
```

Running the suggested command initializes git hooks and generates a configuration file.
```bash
$ bundle exec lefthook install -f
Added config:  /home/jeremiah/boxspring/lefthook.yml
```

The default file contains examples for integrating several popular tools, but it can all be replaced with this simple entry.
```yaml
pre-commit:
  parallel: true
  commands:
    standard:
      run: bundle exec standardrb
```

# Commit

Configuring Standard is a complete, though small, unit of work. After staging changes, four files are ready to commit.
```bash
$ git add -A
$ git status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
	new file:   .ruby-version
	new file:   Gemfile
	new file:   Gemfile.lock
	new file:   lefthook.yml
```

I mentioned the intentional single-quotes in the Gemfile earlier, and this is where they come up again. After attempting to commit these changes, Standard will insist that those quotes be standardized[^3].
```bash
$ git commit
Lefthook v0.7.7
RUNNING HOOKS GROUP: pre-commit

  EXECUTE > standard
 standard: Use Ruby Standard Style (https://github.com/testdouble/standard)
standard: Run `standardrb --fix` to automatically fix some problems.
  Gemfile:5:16: Style/StringLiterals: Prefer double-quoted strings unless you need single quotes to avoid extra backslashes for escaping.


SUMMARY: (done in 1.06 seconds)
🥊  standard
```

Conveniently, Standard has the ability to fix most things automatically using the `--fix` flag.
```bash
$ bundle exec standardrb --fix
$ git diff
diff --git a/Gemfile b/Gemfile
index 2204001..cf01ab8 100644
--- a/Gemfile
+++ b/Gemfile
@@ -2,7 +2,7 @@

 source "https://rubygems.org"

-ruby File.read('.ruby-version').chomp
+ruby File.read(".ruby-version").chomp

 group :development do
   gem "lefthook"
```

Note that it's still necessary to __add these changes__ before committing. Standard only monitors the current state of the file, not the staged version. This is honestly a bit of a rough spot in an otherwise smooth process[^4].
```
$ git add -A
$ git commit
```

## Final Thoughts

Maintaining code is largely an exercise in communication, both with collaborators and with your future self. These simple steps help me to reduce friction in both cases, and ensure that I can always pick up where I left off in any given project.

The current [boxspring](https://github.com/allknowingfrog/boxspring) code is available on GitHub.

## Footnotes

[^1]: It will be a foundation that holds something RESTful, and when it comes to project names, I'll always take clever over explanatory.
[^2]: If you're writing Ruby, you should be using a version manager, and if you're using a version manager, I recommend [rbenv](https://github.com/rbenv/rbenv).
[^3]: Standard's name is both clever _and_ explanatory.
[^4]: Though apparently not rough enough to motivate me to solve it.
