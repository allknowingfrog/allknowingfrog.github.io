---
layout: post
title:  "nav_to (Rails Helper for Bootstrap Navs)"
date:   2019-06-21 10:00:00 -0500
---

I like using Bootstrap navs as the primary navigation tool in my Rails projects, but keeping track of the active member of a nav can be tricky. In the past, I've experimented with setting tabs based on the current controller, but I always end up coding for various corner cases. My current approach pairs a helper with a simple tab-tracking variable.

I initialize a `@tabs` hash in my application controller and use it to track key-value pairs where the key represents a nav and the value names a tab within it. In this example, the books controller tells the `:application` nav to set `:books` as the active tab. Within the books area, a secondary nav is set to select `:all`.

```ruby
# app/controllers/books_controller.rb
class BooksController < ApplicationController
  before_action :set_tabs
  ...
  private
    def set_tabs
      @tabs[:application] = :books
      @tabs[:books] = :all
    end
```

Rather than add a conditional `.active` class on every nav item, I wrap a `nav_to` helper around `link_to`. It replaces the first argument with a key and a value to check in the `@tabs` array. It also eliminates some redundancy in setting Bootstrap classes.

```ruby
# app/helpers/application_helper.rb
module ApplicationHelper
  def nav_to(nav, tab, path, options={})
    options[:class] = 'nav-item nav-link'
    options[:class] << ' active' if @tabs[nav] == tab
    label = options.key?(:label) ? options[:label] : tab.to_s.titleize
    link_to label, path, options
  end
  ...
```

Usage is pretty simple.
```ruby
# app/views/layout/application.html.haml
...
.navbar-nav
  = nav_to :application, :books, books_path
  = nav_to :application, :authors, authors_path, label: 'Writers'
...
```

```ruby
# app/views/layouts/books.html.haml
...
.navbar-nav
  = nav_to :books, :all, books_path
  = nav_to :books, :staff_picks, staff_picks_path
...
```

If other people were interested, I might even consider turning this into a gem. It would certainly DRY up my collection of side projects.
