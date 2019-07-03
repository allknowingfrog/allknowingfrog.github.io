---
layout: post
title:  "Rails Transaction Abstraction"
date:   2019-07-03 14:00:00 -0500
---

I use a database transaction at some point in virtually every Rails project I tackle. If you aren't aware, a transaction forces a group of queries to succeed or fail together. If any ActiveRecord object in the series throws an exception, all changes are rolled back for all records involved ([documentation](https://api.rubyonrails.org/classes/ActiveRecord/Transactions/ClassMethods.html)). The classic example is debit-credit accounting, which always inserts two records for every transaction.

```ruby
ActiveRecord::Base.transaction do
  debit.save!
  credit.save!
end
```

Note the bang methods. A regular validation error will not trigger a rollback. We have to elevate validation errors to exceptions, which means we'll need to rescue those exceptions. Plus, we need to track the success of the operation and report back to the user.

```ruby
begin
  ActiveRecord::Base.transaction do
    debit.save!
    credit.save!
  end
  success = true
rescue ActiveRecord::RecordInvalid
  success = false
end
```

That's a lot of boilerplate, but we can DRY it out.

```ruby
class ApplicationController < ActionController::Base
  private
    def with_transaction
      ActiveRecord::Base.transaction { yield }
      true
    rescue ActiveRecord::RecordInvalid
      false
    end
```

```ruby
# some update action
success = with_transaction do
  debit.save!
  credit.save!
end
```

### A better transactional save

To use `save!`, as in the previous examples, attributes must be set in a separate, preceeding step. In practice, a simple update action may use a params method to update a record in one step, plus check the return value to redirect or render errors. Our hypothetical accounting application might have an accounts controller, for example.

```ruby
class AccountsController < ActionController::Base
  ...
  def update
    if @account.update(account_params)
      redirect_to accounts_url
    else
      render :edit
    end
  end
```

Even with our new function, a transactional update is pretty verbose.

```ruby
class TransfersController < ActionController::Base
  ...
  def update
    @debit.amount = params[:amount]
    @credit.amount = params[:amount]

    if @debit.valid? && @credit.valid?
      success = with_transaction do
        @debit.save!
        @credit.save!
      end
    else
      success = false
    end

    if success
      redirect_to transfers_url
    else
      render :edit
    end
  end
```

Notice that manual validation check before the transaction? Without it, an earlier valid record can trigger a query that will be rolled back by a later invalid record. If we can determine that a record is invalid at the application level, we probably want to avoid hitting the database at all. Let's extract another function.

```ruby
class ApplicationController < ActionController::Base
  private
    ...
    def transactional_save(objects)
      if objects.all?(&:valid?)
        with_transaction do
          objects.each { |o| o.save! }
        end
      else
        false
      end
    end
```

```ruby
class TransfersController < ActionController::Base
  ...
  def update
    @debit.amount = params[:amount]
    @credit.amount = params[:amount]

    if transactional_save([@debit, @credit])
      redirect_to transfers_url
    else
      render :edit
    end
  end
```

We still have the extra step to set attributes, but I can live with that.

### What goes up...

Transactionally destroying multiple objects is a slightly different problem. We don't need validation, but we do need to catch a different exception. We can modify `with_transaction` slightly to accomodate this and then write one more simple function. Here's the final version of the code:

```ruby
class ApplicationController < ActionController::Base
  private
    def with_transaction(e=ActiveRecord::RecordInvalid)
      ActiveRecord::Base.transaction { yield }
      true
    rescue e
      false
    end

    def transactional_save(objects)
      if objects.all?(&:valid?)
        with_transaction do
          objects.each { |o| o.save! }
        end
      else
        false
      end
    end

    def transactional_destroy(objects)
      with_transaction(ActiveRecord::RecordNotDestroyed) do
        objects.each { |o| o.destroy! }
      end
    end
end
```

Because we have no attributes to set, a transactional destroy action is nearly as compact as any other destroy.

```ruby
class TransfersController < ActionController::Base
  ...
  def destroy
    if transactional_destroy([@debit, @credit])
      redirect_to transfers_url
    else
      render :edit
    end
  end
```

These simple convenience functions are not intended to be a universal solution to transactional operations, but they address most of my needs. As always, if you think I'm doing it wrong, please feel free to [tell me](https://twitter.com/allknowingfrog).
