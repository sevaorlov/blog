---
layout: post
title: Preload polymorphic associations associations
date: 2016-09-18 13:09
categories:
---

Have you ever had N+1 query problem with polymorphic associations associations? Let me explain what I mean :)

For example, there is a model `View` with polymorphic `viewable` association:

{% highlight ruby %}
class View < ActiveRecord:Base
  belongs_to :viewable, polymorphic: true
end
{% endhighlight %}

`Viewable` can be either a `Community` or `Content` and `Content` also have a polymorphic association `composite`:

{% highlight ruby %}
class Community < ActiveRecord::Base
end

class Content < ActiveRecord::Base
  belongs_to :composite, polymorphic: true
end
{% endhighlight %}

`Composite` can be either a `AudioFile` or `VideoFile`:

{% highlight ruby %}
class AudioFile < ActiveRecord::Base
end

class VideoFile < ActiveRecord::Base
end
{% endhighlight %}

Starting from Rails 3 there is an ability to load all `views` with all `viewables` with this code:

{% highlight ruby %}
View.includes(:viewable).all
{% endhighlight %}

That is cool, but what about composites? They will not be loaded and you will have N+1 query for each `Content`
in database. If all `viewable` types have `composite` association - that is not a problem, you can load them too with:

{% highlight ruby %}
View.includes(viewable: :composite).all
{% endhighlight %}

But in our case this code will fail with:

{% highlight ruby %}
ActiveRecord::AssociationNotFoundError: Association named 'composite' was not found on Community; perhaps you misspelled it?
{% endhighlight %}

I did not know about that, but there is a public API for preloading any association you want in Rails. I have come
across it in Rails code - [ActiveRecord::Associations::Preloader](http://apidock.com/rails/v4.2.7/ActiveRecord/Associations/Preloader/preload).
Never heard of it, but it is used for preloading associations in `ActiveRecord::Base#preload`.

In order to load all `views` with `viewables` and all `contents` with `composites` you need to instantiate preloader
and call `preload` method with desired association in Rails 4.2.7:

{% highlight ruby %}
# rails 4.2.7
views = View.includes(:viewable)
content_views = views.select { |view| view.viewable.is_a? Content }
preloader = ActiveRecord::Associations::Preloader.new
preloader.preload content_views, { viewable: :composite }
puts views
{% endhighlight %}

In Rails 3.2.22 you need to pass these arguments to preloader `new` method and call `run`:

{% highlight ruby %}
# rails 3.2.22
views = View.includes(:viewable)
content_views = views.select { |view| view.viewable.is_a? Content }
preloader = ActiveRecord::Associations::Preloader.new content_views, { viewable: :composite  }
preloader.run
puts views
{% endhighlight %}

Yes, we need to pass an array to preloader, so that this approach may not be worth it in some cases. You need to decide
what is better in your circumstances - whether to have more database requests or a loop through an array.

This way Rails will send one request to fetch all views, then one request to fetch all `Community` `viewable`'s, one
request for all `Content` `viewable`'s and one request for each `composite` type.

{% highlight sql %}
View Load (0.1ms)  SELECT "views".* FROM "views"
Content Load (0.1ms)  SELECT "contents".* FROM "contents" WHERE "contents"."id" IN (1, 2, 3)
Community Load (0.1ms)  SELECT "communities".* FROM "communities" WHERE "communities"."id" IN (1, 2, 3)
AudioFile Load (0.1ms)  SELECT "audio_files".* FROM "audio_files" WHERE "audio_files"."id" IN (1, 2, 3)
{% endhighlight %}

To be honest you will have the same number of requests if all `Content` `viewable`'s have different `composite` types,
but if you have 100 `composite`'s of the same type - you will have 99 database requests less, that can be really
helpful.

The script I wrote for reproducing the problem and the solution:

{% highlight sql %}
begin
  require 'bundler/inline'
rescue LoadError => e
  $stderr.puts 'Bundler version 1.10 or later is required. Please update your Bundler'
  raise e
end

gemfile(true) do
  source 'https://rubygems.org'
  # gem 'activerecord', '~> 3.0'
  gem 'activerecord', '~> 4.0'
  gem 'sqlite3'
  gem 'minitest'
end

require 'active_record'
require 'minitest/autorun'
require 'logger'

# Ensure backward compatibility with Minitest 4
Minitest::Test = MiniTest::Unit::TestCase unless defined?(Minitest::Test)

ActiveRecord::Base.establish_connection(adapter: 'sqlite3', database: ':memory:')
ActiveRecord::Base.logger = Logger.new(STDOUT)

ActiveRecord::Schema.define do
  create_table :views, force: true do |t|
    t.references :viewable, polymorphic: true
  end

  create_table :communities, force: true do |t|
  end

  create_table :contents, force: true do |t|
    t.references :composite, polymorphic: true
  end

  create_table :audio_files, force: true do |t|
  end

  create_table :video_files, force: true do |t|
  end
end

class View < ActiveRecord::Base
  belongs_to :viewable, polymorphic: true
end

class Community < ActiveRecord::Base
end

class Content < ActiveRecord::Base
  belongs_to :composite, polymorphic: true
end

class AudioFile < ActiveRecord::Base
end

class VideoFile < ActiveRecord::Base
end

class Test < Minitest::Test
  def test_preloading
    10.times do
      View.create viewable: Content.create(composite: AudioFile.create)
    end

    20.times do
      View.create viewable: Community.create
    end

    views = View.includes(:viewable)
    content_views = views.select { |view| view.viewable.is_a? Content }

    # 4.2.7
    preloader = ActiveRecord::Associations::Preloader.new
    preloader.preload content_views, { viewable: :composite }

    # 3.2.22
    # preloader = ActiveRecord::Associations::Preloader.new content_views, { viewable: :composite }
    # preloader.run

    puts views
  end
end
{% endhighlight %}
