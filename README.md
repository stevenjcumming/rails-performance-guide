# Rails Performance Guide

## Overview
Disclaimer: This performance guide isn't intended to be exhaustive 

Use this guide as a foundation for performance improvements and developing with performance in mind. Performance fine-tuning is best done with monitoring, profiling, and experimentation. Aim for 100 ms, stay under 200 ms, and 500+ ms requires action. Before you start making improvements, identify where the biggest problem is and where the biggest impact could be.  

**Definitions**
- _Never_: Never means never
- _Don't_: Don't unless you have a really really good reason
- _Avoid_: Avoid unless you have a good reason

**Gems**
* [rack-mini-profiler](https://github.com/MiniProfiler/rack-mini-profiler) - Profiler for your development and production Ruby rack apps
* [Rack::Attack!!](https://github.com/kickstarter/rack-attack) - Rack middleware for blocking & throttling
* [Bullet](https://github.com/flyerhzm/bullet) - help to kill N+1 queries and unused eager loading
* [Active Record Doctor](https://github.com/gregnavis/active_record_doctor) - Identify database issues before they hit production
* [lol_dba](https://github.com/plentz/lol_dba) - Scan your application models and displays a list of columns that probably should be indexed
* [PgHero](https://github.com/ankane/pghero) - A performance dashboard for Postgres
* [Fasterer](https://github.com/DamirSvrtan/fasterer) - Static analysis that checks speed idioms from Fast Ruby
* [pg_search](https://github.com/Casecommons/pg_search) - pg_search builds ActiveRecord named scopes that take advantage of PostgreSQL’s full text search
* [Elasticsearch](https://github.com/elastic/elasticsearch-rails) - Elasticsearch integrations for ActiveModel/Record and Ruby on Rails

**TOC**
* [Caching](#caching)
* [Background Jobs](#background-jobs)
* [ActiveRecord](#activerecord)
* [Database](#database)
* [Puma](#puma)
* [Misc](#misc)

## Caching

### Guidelines
- Cache view renderings whenever possible. The closer to the user, the more performant
- Generic data can be cached for everyone
- Nested caching (except for memoization) allows you to keep valid cache when data changes. 
- Large (or slow), frequently-accessed, and infrequently-updated data offers the biggest impact. 

### Cache Store

There are a few different options for choosing a cache store and you can read more about them in the Rails caching [guide](https://guides.rubyonrails.org/v7.0/caching_with_rails.html#cache-stores). In most cases you can start a project with MemoryStore (1 server) and switch to Redis (2 or more servers) as your infrastructure grows. If you are unafraid of the costs you can go straight to Redis. 

### Caching in Practice

Rails makes caching pretty easy and there are three techniques worth mentioning: _Russian doll_, _memoization_, and _low-level caching_

#### Russian Doll Caching

[Russian doll caching](https://guides.rubyonrails.org/v7.0/caching_with_rails.html#russian-doll-caching) is nested cached fragments inside other cached fragments. The advantage of Russian doll caching is that if a single product is updated, all the other inner fragments can be reused when regenerating the outer fragment.

```erb
<%# views/users/index.html.erb %> 
<% cache ['users', @users.map(&:id), @users.maximum(:updated_at).to_i] do %>
  <%= render partial: 'users/user', collection: @users, cached: true %>
<% end %>

<%# views/users/_user.html.erb %>
<% cache user do %>
  <p>Name: <%= user.name %></p>
  <p>Email: <%= user.email %></p>
  <p>Location: <%= user.location %></p>
  <p>Comments:</p>
  <%= render partial: "users/comment", collection: user.comments, as: :comment, cached: true %>
  <p>Attachments:</p>
  <%= render partial: "users/attachment", collection: user.attachments, as: :attachment, cached: true %>
<% end %>
```

```ruby
# views/api/v1/users/index.json.jbuilder
json.cache! ["v1", 'users', @users.map(&:id), @users.maximum(:updated_at).to_i] do
  json.users @users, partial: "user", as: :user, cached: true
end

# views/api/v1/users/_user.json.jbuilder
json.cache! ["v1", user] do
  json.extract! user, :name, :email, :location
  json.comments user.comments, partial: "api/v1/comments/comment", as: :comment, cached: true
  json.attachments user.attachments, partial: "api/v1/attachemnts/attachment", as: :attachment, cached: true
end
```

The nesting looks like `["v1", 'users', @users.map(&:id), @users.maximum(:updated_at).to_i]` then `["v1", user]` then collection caching for `comments` and `attachments`

The top level cache tells Rails if there's a new/removed user from the list OR if a user is updated. It's also needed for any changes to comments or attachments. If we just used ['v1', 'users'] changes to a comment wouldn't be rendered. We can achieve this with the `touch` method like this: `belongs_to :user, touch: true`. This updates `updated_at` on a comment and the user it belongs to, and expires the cache. 

#### Memoization

Simply memoization is calling the same method and doing the calculations once. Memoization is contained within a single request. The most common use for memoization is `current_user`. 

```ruby
def current_user
  @current_user ||= User.find(params[:user_id])
end
```
You can also use a begin block 

```ruby
def current_user
  @current_user ||= begin
    ... Expensive calculations ...
    User.find(params[:user_id])
  end
end
```

Nested memoization can lead to bad data, so only use memoization at the lowest level. Methods that use changing paramaters are not good candidates for memoization. Memoization also ignores `nil` and `false`, which can cause re-calculations. The "work around" is to return early if the variable is defined: 

```ruby
def user
  return @user if defined?(@user)
  @user ||= User.find(params[:user_id])
end
```

#### Low-level Caching

Low-level caching means you are responsible for fetching, keys, and expiration. This strategy requires more work and caution, because of the fine grain control. You'll need to consider how volatile the data it is and the consequences of stale data. 

The Rails caching guide provides and example with an external API call, which is a common use case

```ruby
class Product < ApplicationRecord
  def competing_price
    Rails.cache.fetch("#{cache_key_with_version}/competing_price", expires_in: 12.hours) do
      Competitor::API.find_price(id)
    end
  end
end
```

If you aren't using `expires_in` or `expires_at` then you will manually have to expire the entry with `Rails.cache.delete()`

A few other use cases for low-level caching:
- Computationally intensive operations - Example: Data aggregation with heavy calculations for a sales report
- User-specific data - Example: User recommendations based on browsing history
- Expensive database queries - This should be complex or slow database queries that are executed frequently

This strategy add more complexity, it doesn't have to be a last resort, but use it prudently. If you search `cache.fetch` on Github it's not a coincidence that all the code is for GitLab, Discourse, Mastodon, and Forem.  

## Background Jobs

### Backends

Active Job is the built-in framework for creating, enqueuing and executing background jobs. You have several options for a queuing backend. Here are some options:

- [Sidekiq](https://github.com/sidekiq/sidekiq/wiki/Active-Job) - Very popular and very fast, but it's possible you could lose data (especially if it's your Redis instance isn't configured correctly).
- [DelayedJob](https://github.com/collectiveidea/delayed_job) - Very reliable and cheaper, but not as performant as a Sidekiq. Although it should be good enough for most applications. Companies like Shopify, GitHub, and Betterment have used DJ for their jobs. 
- [GoodJob](https://github.com/bensheldon/good_job) - GoodJob is a multithreaded, Postgres-based, ActiveJob backend for Ruby on Rails.

### What to move to the background

- Sending emails
- Warming caches
- Updating search indexes 
- Updating user preferences for recommendations
- Sending webhooks
- External API calls (sometimes)
- Any thing the user doesn't care about in the request/response cycle
- Generating invoices
- Transcoding a video (although I'd recommend using a 3p service)

In some cases when background job performance is reach its limiit, you may need to switch from delayed job to sidekiq. If you still need more performance, you may even consider using sidekiq directly. 

## Database

- Add indexes to columns used in WHERE clauses. Start with most frequently used and work towards less frequently used. You don't want to over index. 
- Compound indexes can be useful when two columns are always referenced together such as on Join Tables or polymorphic associations. 
- `find_each` and `in_batches` allows you to process in batches (1000 by default) which can be a HUGE memory saver for sufficiently large tables. It's _imporant_ to know that if you require custom ordering, you'll have to implement batching yourself. [source](https://api.rubyonrails.org/classes/ActiveRecord/Batches.html)
- N+1 queries can be eliminated with `:includes` 
- Preload rails scopes [Justin Weiss's blog post](https://www.justinweiss.com/articles/how-to-preload-rails-scopes/)
- Use `size` instead of `count` for Active Record Relations
- Use !! instead of .present? (except for checking empty strings)
- Use `exists?` on Active Record Relations only if you don't use the relation later. `exists?` always hits the database. 
- Use `.load.any?` for `if @users.load.any?; @users.each ...` 
- `if @users.any?; @users.first(3).each ...` is ok if you only use a portion of the relation 
- SELECT only the columns you are going to use `User.select(:id, :name, :email)`
- Use [ActiveRecord::Calculations](https://api.rubyonrails.org/classes/ActiveRecord/Calculations.html) for math operations (e.g. sum, minimum, maximum, average). The larger the dataset the more beneficial it will be. 
- In some cases it's appropriate to reduce the number of SQL queries by consolidating one-to-one relationships into a single table. An overly simplistic example is having a separate table for emails when a user only has one email. 

#### Special Note

Not all N+1 queries are the same. With Russian doll caching you can cache small, simple queries for each component. With one big SQL query that joins multiple tables you will incur that cost when cache gets busted. Your caching strategy MUST build incrementaly i.e. the a new record create a new cache vs destroy and re-creating when a new record is added to the collection.

"We have N+1 up the _wazoo_ for chat (at Basecamp)." - DHH [source](https://www.youtube.com/watch?v=ktZLpjCanvg&t=241s)

## Puma

With Puma we need to set 3 settings in our configuration file (config/puma.rb). Each setting will depend on your server (or container). 

1. Worker count
2. Thread count
3. Copy-on-write

```ruby
# config/puma.rb

# Best practice is to set the variable from the server
# WEB_CONCURRENCY = (TOTAL_RAM / (RAM_PER_PROCESS * 1.2))
# Use an upper limit of x1.5 the number of available hyperthreads (vCPUs) 
# For a t3.small workers = 3 
# It's generally accepteable to start with 3 workers for 1 GB of RAM
workers ENV.fetch("WEB_CONCURRENCY") { 3 }

# for MRI/C Ruby use 5 for min and max
# for JRuby you need to experiment by increasing them until you run out of memory or CPU resources.
threads 5, 5

# Copy-on-write
preload_app!

# include the following for rails Rails 4.1 to 5.2
on_worker_boot do
  ActiveRecord::Base.establish_connection
end
```

## Misc

- For Russian doll caching it's benefit to switch the order to descending like this:
```ruby
add_index(:users, :updated_at, order: {updated_at: "DESC NULLS LAST"})
```
- Use pagination where appropriate, and it's often better to use a gem. 
- Don't use `like` with front wildcard `User.where("email like ?","%@gmail.com%")`. This will check the entire table. 
- Use 3p solutions for full text search like pg_search, Sphinx, or Elasticsearch 
- Use the collection rendering when appropriate 
- Avoid exceptions as control flow 
- Don't optimize too early. Develop with performance in mind, but over-optimization can add unnecessary complexity for a site with little traffic. 

**Additional Resources**
* [Caching with Rails](https://guides.rubyonrails.org/caching_with_rails.html)
* [New Relic](https://newrelic.com/)
* [Skylight](https://www.skylight.io/)
* [Sentry](https://sentry.io/for/rails/)
* [Speedshop](https://www.speedshop.co/)
* [Nate Berkopec's interview with David Heinemeier Hansson](https://www.youtube.com/watch?v=ktZLpjCanvg)
* [How Basecamp Next got to be so damn fast without using much client-side UI](https://signalvnoise.com/posts/3112-how-basecamp-next-got-to-be-so-damn-fast-without-using-much-client-side-ui)
