# Arcane

[![Build Status](https://travis-ci.org/cloudsdaleapp/arcane.png?branch=master)](https://travis-ci.org/cloudsdaleapp/arcane)
[![Coverage Status](https://coveralls.io/repos/cloudsdaleapp/arcane/badge.png)](https://coveralls.io/r/cloudsdaleapp/arcane)
[![Code Climate](https://codeclimate.com/repos/51e6d15913d6373751014a5a/badges/c2f7e13ed4e4f06be966/gpa.png)](https://codeclimate.com/repos/51e6d15913d6373751014a5a/feed)
[![Gem Version](https://badge.fury.io/rb/arcane.png)](http://badge.fury.io/rb/arcane)
[![Dependency Status](https://gemnasium.com/cloudsdaleapp/arcane.png)](https://gemnasium.com/cloudsdaleapp/arcane)

Easy to use parameter management, extending [Strong Parameters](https://github.com/rails/strong_parameters),
making them a first class citizen. Arcane provides you with helpers which guide you in leveraging regular
Ruby classes and object oriented design patterns to organise and easily harnass the power of strong parameters
in Rails 3 and 4.

Arcane magic is real and reliable, no cheap tricks.
Inspired by [Pundit](https://github.com/elabs/pundit)

#### Breaking Changes - Check [CHANGELOG.md](https://github.com/cloudsdaleapp/arcane/blob/master/CHANGELOG.md)

#### Is it production ready? Yes.

## Installation

Add this line to your application's Gemfile:

```ruby
gem "arcane", "~> 1.0.0"
```

If you are still on Rails 3, add this to your Gemfile as well:

```ruby
gem "strong_parameters", "~> 0.2.0"
```

And then execute:

```bash
$ bundle
```

To install arcane, execute:

```bash
rails g arcane:install
```

## Usage

This is how easy it is to use:

```ruby
def create
  @article = Article.new params.for(Article).refine
end

def update
  @article.find(params[:id])
  @article.update_attributes params.for(@article).as(current_user).refine
end

def destroy
  @article.find(params[:id])
  @article.update_attributes params.for(@article).on(:destroy).refine
end
```

### Include the Arcane Module

Though, we need to do a couple of things before we can get started. First of all include `Arcane` in your
controller. This will extend strong parameters with all the arcane methods you saw above in the example.

```ruby
# app/controllers/application_controller.rb
class ApplicationController < ActionController::Base
  include Arcane
end
```

### Create your first Refinery

Before you can use the parameter methods, you need a Refinery for the model you want to pass parameters to.
Simply create the directory `/app/refineries` in your Rails project. Create a Refinery your model, in this
case Article. `/app/refineries/article_refinery.rb`. Create a class in that file called `ArticleRefinery`.
You can also use the rails generator below:

```bash
rails g arcane:refinery article
```

Methods defined in the refinery should reflect the controller method for clarity, but can be anything you
want it to be. These methods must return an array containing the same parameters you would otherwise send
to strong parameters.

It can be initiated using a `Struct` which accepts an `object` and a `user`. Arcane will automatically send
`current_user`, if present to the refinery as well as the object you want to apply the parameters on.

```ruby
# app/refineries/article_refinery.rb
class ArticleRefinery < Struct.new(:object,:user)
  def create
    [:title] + update
  end
  def update
    [:content]
  end
end
```

**Note**, for convenience sake there is a class called `Arcane::Refinery` which you can use instead of
the struct. This class gives you the basic functionality of a refinery for free.

```ruby
class CommentRefinery < Arcane::Refinery; end
```

### Using Arcane in your controller

Next up, using the Arcane methods. There first three are; `for`, `as`, `on` and can all be called on
an instance of rails params, chained, and in any order you want. The fourth one, `refine` you call to
pull your parameters through a refinery.

* `for` - The model or object you want to filter parameters
* `as` - The user performing the action, by default `current_user`
* `on` - The action or rather, refinery method that is called

* `refine` - Wraps everything up and finds your desired filter.

```ruby
refined_params = params.for(@article).as(current_user).on(:update).refine
```

Here's an example of how a controller can look with Arcane paramters.

```ruby
class GameController < ApplicationController
  def create
    @game = Game.create(params.for(Game).as(user_from_location).refine)
  end

  def update
    @game.find(params[:id])
    @game.update_attributes params.for(@game).as(current_user).refine
  end

  def update_many
    @games = Game.find(params[:ids])
    @games.each do |game|
      game.update_attributes(params.for(@game).on(:update).refine)
    end
  end

private

  def user_from_location
    # ...
  end
end
```

## Features

### Invokable anywhere.
As arcane extends ActionController::Parameters you can invoke in anywhere and start toying around with
the arcane methods. This is good if you have some other way of getting data in to your application outside
the context of a controller.

```ruby
@user, @post = User.find(1), Post.find(24)

my_params = ActionController::Parameters.new({ post: { content: "Hello" } })
my_params.for(@post).as(@user).on(:create)
```

If you want to delegate some behaviour to your model, you can always create your own constructor.

```ruby
class Post < ActiveRecord::Base
  def self.refined_create(params)
    create(params.for(self).refine)
  end
end

class PostsController
  def create
    @post = Post.refined_create(params)
  end
end
```

### Automatic method detection.
If you have specified no refinery action in your chain to params, Arcane tries to find out for itself
what method to use. Arcane uses the action key in the rails parameters to determine the refinery method.

```ruby
class CommentsController < ApplicationController
  def update
    @comment = Comment.find(params[:id])
    @comment.update_attributes params.for(@comment)
  end
end

class CommentRefinery < Arcane::Refinery
  def update
    [:email,:name,:text]
  end
end
```

### Easily manage conditional parameters
As an application grows larger, you usually want to filter what parameters based on some rules, usually
by checking the role of your current user or the state of the object you want to set the parameters on.
Arcane easily helps you do this.

```ruby
class UserRefinery < Arcane::Refinery
  def update
    params = [ { abuse_reports: AbuseReportRefinery.new(AbuseReport.new,user).create } ]
    params += [:suspended]   if user.admin?
    params += [:name,:email] if user.admin? or user == object
  end
end
```

### Default parameters.
You are able to specify a `default` method in your refinery which will be prioritized if no the method
you call does not exist. If default is not specified it will be as the refinery returned an empty array.

```ruby
class AmbiguityRefinery < Arcane::Refinery
  def default
    [:data]
  end
end
```

### Custom root requirement.
You are able to disable or change the root requirement. Let's say you have a sessions endpoint where
you don't have your username and password parameters wrapped in a root. Now you can use the root class
method and set it to nil or false and it will automatically not require it.

```ruby
class SessionRefinery < Arcane::Refinery
  def self.root
    false
  end
end
```

Or if you have a `MeRefinery` for allowing certain parameters on a `/me.json` which is still a user
object. You can just set a root class method on your refinery and it will use this to determine if
the requirements.

```ruby
class MeRefinery < UserRefinery
  def self.root
    :user
  end
end
```

### Refinery inheritence.
Say you have quite similar needs between two different models, one of them might even have inherited
from the other. As arcane's refineries are just regular ruby models you can easily inherit from one
to another and it will just work.

```ruby
class Square
  attr :height, :width
end

class Cube < Square
  attr :depth
end

class SquareRefinery
  def create
    [:height,:width]
  end
end

class CubeRefinery
  def create
    [:depth] + super
  end
end
```

### Building HATEOAS API templates
As your refinery is a class that accepts an object and a user you can invoke it on it's own. This is
really good if you want your application to generate endpoints under different scenarios dynamically.

```ruby

track_params = TrackRefinery.new(Track.new,User.new).create
# => [:title,:description,:duration]

track_url("{track}") + "?" + "{#{track_params.join(",")}}"
# => "http://www.music.com/tracks/{track}?{title,desc,duration}"
```

## Requirements

Currently this gem is only supported for Rails and with any of these ruby versions:

* ruby-2.0.0
* ruby-1.9.3
* ruby-1.9.2
* jruby-19
* rbx-19

## To-do

- [x] Explain Arcane::Refinery
- [x] Write rails generators
- [x] List features
- [x] Add Documentation
- [x] Add documentation for HATEOAS
- [x] Automatic method detection
- [-] RSpec helpers to test Arcane
- [-] Configuration

## Contributing

#### 1. Fork us
#### 2. Create your feature branch

```bash
git checkout -b my-new-feature
```

#### 3. Running specs

```bash
# Rails 3
env BUNDLE_GEMFILE=./gemfiles/rails3.gemfile bundle exec rspec
# Rails 4
env BUNDLE_GEMFILE=./gemfiles/rails4.gemfile bundle exec rspec
```

#### 4. Commit your changes
```bash
git commit -am 'Add some feature'
```

#### 5. Push to the branch
```
git push origin my-new-feature
```

#### 6. Create new Pull Request
