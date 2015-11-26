---
layout: post
title:  "Keeping your Chef DRY"
date: 2015-11-26 11:33:00
categories: devops
---

Like many things Ruby, [Chef][] comes with its own DSL which is both a blessing
and a curse. It's great to keep the barriers to entry low (and you can make
things look pretty...) but sometimes it makes more advanced techniques quite
difficult.

[Rails][] makes some great choices in keeping items (models, controllers) as 
Ruby classes and not attempting to hide that from the coder but it assumes a
level of familiarity with programming that may scare some people (it shouldn't!
There are many [great resources][guide] online to help people learn) whereas
Chef tries to hide the Ruby side of things behind their own
[DSL][chef_recipe_dsl].

Keep things DRY
===

When looking to abstract common code to keep things [DRY][] it's common
practice to create helper modules and include them when necessary.  For example
if you wanted to include the method `sort` in both your CommentsController and
PostsController you can do the following:

{% highlight ruby %}
module Sortable
  def sort
    "updated_at DESC"
  end
end

class CommentsController
  include Sortable
end

class PostsController
  include Sortable
end
{% endhighlight %}

So now you can use `sort` in instances of CommentsController or PostsController
without having to re-define it in each. Awesome.

The problem with Chef recipes
===

That's great for platforms like Rails that separate each controller or model in
to its own class.  You can mix in behaviour to the controllers that need it
while not worrying about polluting the ones that do not.  Chef recipes,
unfortunately, are all instances of the same `Chef::Recipe` class.

The example provided by the official [windows cookbook][] on how to include
helper libraries in to your recipes is to use
`::Chef::Recipe.send(:include, Windows::Helper)`.
I don't like this for two reasons:

1. It uses `#send` to bypass restrictions on calling the private method
`#include`
2. It monkey patches ALL instances of the Chef::Recipe class so you risk 
modifying behaviour of other recipes in the run list if they have a different
definition of the same method.

Here's an example:

{% highlight ruby %}
# with_library/libraries/helper.rb
module WithLibrary
  module Helper
    def example_method
      puts "Do stuff here"
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
# with_library/recipe/default.rb
Chef::Recipe.send(:include, WithLibrary::Helper)
puts "with_library #{ self }: #{ self.methods.grep(/example/) }"
example_method
{% endhighlight %}

{% highlight ruby %}
# without_library/recipe/default.rb
puts "without_library #{ self }: #{ self.methods.grep(/example/) }"
example_method
{% endhighlight %}

Then by running `chef-client -z -r with_library,without_library` we get the
following output:

    Compiling Cookbooks...
    with_library #<Chef::Recipe:0x57ff480>: [:example_method]
    Do stuff here
    without_library #<Chef::Recipe:0x59dfc78>: [:example_method]
    Do stuff here
    Converging 0 resources

So we can see that our `Chef::Recipe` instances have two different object IDs
(0x57ff480 and 0x59dfc78 here, yours may differ) yet they both have
`#example_method` instance methods.  Oops.  Even worse, if you change the run
list order to be without_library,with_library it'll throw an exception:

    NameError
    ---------
    No resource, method, or local variable named `example_method' for
    `Chef::Recipe "default"'

So you're now in the situation that your recipes behave differently 
depending on their run order.

A different way
===

In order to workaround this, I use [Object#extend][object_extend] in my Chef recipe:

{% highlight ruby %}
# with_library/libraries/helper.rb
module WithLibrary
  module Helper
    def example_method
      puts "Do stuff here"
    end
  end
end
{% endhighlight %}

{% highlight ruby %}
# with_library/recipe/default.rb
self.extend(WithLibrary::Helper)
puts "with_library #{ self }: #{ self.methods.grep(/example/) }"
example_method
{% endhighlight %}

{% highlight ruby %}
# without_library/recipe/default.rb
puts "without_library #{ self }: #{ self.methods.grep(/example/) }"
# Don't call example_method here as it doesn't exist!
# example_method
{% endhighlight %}

Now we get the same behaviour for both run orders, and we've only included the
helper method where it's really needed.  Now our `chef-client` run produces the
following:

    Compiling Cookbooks...
    with_library #<Chef::Recipe:0x44cc448>: [:example_method]
    Do stuff here
    without_library #<Chef::Recipe:0x53d7530>: []
    Converging 0 resources

So now we're consistent across run orders and haven't risked overwriting methods
used in all recipes.  Woohoo.

[Chef]: https://chef.io/
[chef_recipe_dsl]: https://docs.chef.io/dsl_recipe.html
[DRY]: https://en.wikipedia.org/wiki/Don%27t_repeat_yourself
[guide]: https://www.railstutorial.org/
[object_extend]: http://ruby-doc.org/core-2.1.2/Object.html#method-i-extend
[Rails]: http://rubyonrails.org/
[windows cookbook]: https://github.com/chef-cookbooks/windows#libraries

*[DRY]: Don't repeat yourself
*[DSL]: Domain-specific language
