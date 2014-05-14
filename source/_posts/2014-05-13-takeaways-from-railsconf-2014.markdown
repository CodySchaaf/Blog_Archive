---
layout: post
title: "Takeaways From RailsConf 2014"
date: 2014-05-13 13:27:11 -0700
comments: true
categories:
---

Once everyone got passed the 'The Death To TDD' hysteria, this years
most influential talks focused heavily on refactoring. This was
my first Rails conference, and while I have alway attempted to write maintainable
and clean code, I was always a little gun-shy when it came to massive refactoring.
When do you break something into a new class, and when is a method too long?

Starting a little simpler, one of my favorite talks was actually a workshop
on day one. It was with [Tute Costa](http://tutecosta.com/), where he took us
through some short examples of some really chaotic code, and after giving us
a few pointers, he set us loose on refactoring it.

##Taming the Flow of Control

We started out with one of my favorite refactoring techniques, mostly because
it is so simple and the least disrupting of the refactoring techniques yet can
still make a huge difference in readability. This is the code he gave us:

```ruby 1-intention-revealing-method https://github.com/tute/refactoring-workshop/blob/master/1-intention-revealing-method/app.rb Link To Full Source

class ProjectsController
  def index
    # When user is admitted at least a week ago we show it's active projects
    if current_user && current_user.created_at < (Time.now - 7*24*3600)
      @projects = current_user.active_projects

    # If not admitted we show some featured projects, and set a marketing flash
    # message when user is new
    else
      if current_user && current_user.created_at > (Time.now - 7*24*3600)
        @flash_msg = 'Sign up for having your own projects, and see promo ones!'
      end
      @projects = Project.featured
    end
  end
end

```

He told us that a simple technique he uses is to just take the comment, and
convert it into the method name that is called by the if statement. Here is what
I did:

```ruby My Revision

class ProjectsController
  def index

    if user_created_more_than_a_week_ago?
      @projects = current_user.active_projects
    else
      @flash_msg = 'Sign up for having your own projects, and see promo ones!'
      @projects = Project.featured
    end
  end

  def user_created_more_than_a_week_ago?
    current_user && current_user.created_at <  weeks_ago(1)
  end

  def weeks_ago(n)
    Time.now - n * 7*24*3600
  end

end

```

As you can see the first long cryptic if statement was easily reduced to a much
more readable sentence-like method. Further more through this process it became
clear that the nested if check wasn't even needed. 

##Polymorphism at its Finest

I had never used polymorphism quite like this before. Instead of constantly checking
the the object we are dealing with can respond to a method, ensured that all
object at this point in the code could respond to the method. Now when I put
it like that it sounds so obvious, I mean isn't that sort of the founding principals
of polymorphism? But what about when we don't find the object we are looking for.
Why shouldn't the same concept apply here. This is the idea that was presented to
use before we started to takle this next part.

```ruby 2-special-case-objects https://github.com/tute/refactoring-workshop/blob/master/2-special-case-objects/app.rb Link To Full Source

class StatusReportJob
  def perform
    users = {}
    User.all.map do |user|
      users[user.name] = {
        name: last_name(user),
        status: last_status(user),
        trial_days: last_trial_days(user)
      }
    end
    users
  end

  private

  def last_name(user)
    if user.last_subscription && user.last_subscription.respond_to?(:name)
      user.last_subscription.name
    else
      'none'
    end
  end

  def last_status(user)
    if user.last_subscription && user.last_subscription.respond_to?(:status)
      user.last_subscription.status
    else
      '-'
    end
  end

  def last_trial_days(user)
    if user.last_subscription && user.last_subscription.respond_to?(:trial_days)
      user.last_subscription.trial_days
    else
      '-'
    end
  end
end

```

The basic idea behind this refactoring was to have a subscription object in preform
that would always be able to respond to the methods called on it. So when
no subscription would have been returned, instead a nullSubscription which respondes
to these methods is returned. Transforming all of that code to this:

```ruby My Revision

class StatusReportJob
  def perform
    users = {}
    User.all.map do |user|
      last_subscription = user.last_subscription
      users[user.name] = {
        name: last_subscription.name,
        status: last_subscription.status,
        trial_days: last_subscription.trial_days
      }
    end
    users
  end
end

```

with a subscription, and nullSubscription defined like this:

```ruby My Revision Classes

class Subscription
  def name
    'Monthly Subscription'
  end
  def status
    'active'
  end
  def trial_days
    14
  end
  def cancel
    # telling payment gateway...
    true
  end
end

class User
  # ...

  def last_subscription
    subscriptions.last || NullSubscription.new
  end

  # ...
end


class NullSubscription
  def cancel
    false
  end

  def name
    'none'
  end

  def status
    '-'
  end

  def trial_days
    '-'
  end
end

```

As you can see last_subscription tries to find your last subscription for you,
but when it fails it returns a new instance of NullSubscription. Then when `last_subscription.name`
is called, we get a `'none'` returned if it dosen't exist, or `'Monthly Subscription'` if it does.

``` ruby

users[user.name] = {
  name: last_subscription.name, # => 'none' or Monthly Subscription
  status: last_subscription.status,
  trial_days: last_subscription.trial_days
}

class Subscription
  def name
    'Monthly Subscription'
  end

  #...

end


class NullSubscription
  def name
    'none'
  end

  #...

end

```

##We Can Rebuild Him, We Have the Technology

Now for some truly horrific code. This is the code you come up with when you are
wrestling to get something to work and you just keep throwing lines at it hoping
something will help. Now it's time to refactor it and make it look like it works as well.

```ruby 3-replace-method-with-method-object https://github.com/tute/refactoring-workshop/blob/master/3-replace-method-with-method-object/app.rb Full Source Code

# 1. Create a class with same initialization arguments as BIGMETHOD
# 2. Copy & Paste the method's body in the new class, with no arguments
# 3. Replace original method with a call to the new class
# 4. Apply "Intention Revealing Method" to the new class. Woot!
class Formatter
  # More code, methods, and stuff in this big class

  def row_per_day_format(file_name)
    file = File.open file_name, 'r:ISO-8859-1'
    # hash[NivelConsistencia][date] = [[value, status]]
    hash = { '1' => {}, '2' => {} }
    dates = []
    str = ''
    CSV.parse(file, col_sep: ';').each do |row|
      next if row.empty?
      next if row[0] =~ /^\/\//
      date = Date.parse(row[2])
      (13..43).each do |i|
        measurement_date = date + (i-13)

        # If NumDiasDeChuva is empty it means no data
        value  = row[7].nil? ? -99.9 : row[i]
        status = row[i + 31]
        hash_value = [value, status]

        dates << measurement_date
        hash[row[1]][measurement_date] = hash_value
      end
    end

    dates.uniq.each do |date|
      if !hash['1'][date].nil? && hash['2'][date].nil?
        # Only 'bruto' (good)
        value = hash['1'][date]
        str << "#{date}\t#{value[0]}\t#{value[1]}\n"
      elsif hash['1'][date].nil? && !hash['2'][date].nil?
        # Only 'consistido' (kind of good)
        value = hash['2'][date]
        str << "#{date}\t#{value[0]}\t#{value[1]}\n"
      else
        # 'bruto' y 'consistido' (has new and old data)
        old_value = hash['1'][date]
        new_value = hash['2'][date]
        str << "#{date}\t#{new_value[0]}\t#{old_value[1]}\t#{old_value[0]}\n"
      end
    end

    str
  end

  # More code, methods, and stuff in this big class
end

```

Now the idea here was to dismantel this method into its components, I gave it
my best but his solution was a little more elagent so I've decided to use his instead.

```ruby Tute Costa's Solution https://github.com/tute/refactoring-workshop/blob/tute-solutions/3-replace-method-with-method-object/app-solution-tute.rb Full Source Code

class FormatAtoB
  SIN_DATOS = -99.9

  def initialize(file_name)
    @file_name = file_name
    @hash  = { '1' => {}, '2' => {} }
    @dates = []
  end

  def perform
    load_format_a_file
    format_data_to_b
  end

  private

  def load_format_a_file
    file = File.open @file_name, 'r:ISO-8859-1'
    CSV.parse(file, col_sep: ';').each do |row|
      next if row.empty? || row[0] =~ /^\/\//
      load_month_in_hash(row)
    end
  end

  def format_data_to_b
    formatted_rows = @dates.uniq.map { |date| formatted_row_for(date) }
    "#{formatted_rows.join("\n")}\n"
  end

  def formatted_row_for(date)
    old_value = @hash['1'][date]
    new_value = @hash['2'][date]
    if dato_bruto?(date)
      day = [date, old_value[0], old_value[1]]
    elsif dato_consistido?(date)
      day = [date, new_value[0], new_value[1]]
    else # 'bruto' y 'consistido'
      day = [date, new_value[0], old_value[1], old_value[0]]
    end
    day.join("\t")
  end

  def load_month_in_hash(row)
    @beginning_of_month = Date.parse(row[2])
    (13..43).each do |i|
      load_day_in_hash(row, i)
    end
  end

  def load_day_in_hash(row, i)
    nivel_consistencia = row[1]
    date = @beginning_of_month + (i - 13)
    @dates << date
    @hash[nivel_consistencia][date] = datos(row, i)
  end

  # If NumDiasDeChuva is empty it means no data
  def datos(row, i)
    data = row[7].nil? ? SIN_DATOS : row[i]
    status = row[i + 31]
    [data, status]
  end

  def dato_bruto?(date)
    @hash['1'][date] && @hash['2'][date].nil?
  end

  def dato_consistido?(date)
    @hash['1'][date].nil? && @hash['2'][date]
  end
end

class Formatter
  def row_per_day_format(file_name)
    FormatAtoB.new(file_name).perform
  end
end

```

There could still be a lot more done here, I mean are you really ever done refactoring?
But I think this grasps the idea, where you want to make use of instance variables
to allow for the method to be broken up without having to pass huge amounts of
variables. It is useful to find the variables that are inherent to the class itself.
Make those your instance variables. From there your goal is simpler, you need to
basically construct these variables to some final product. There is still a lot going
on here, but it definitely looks slightly less overwhelming when looking at manageable chuncks.


##Getting the User Into Shape

For this last part, I need you to imagine a massive user model. This refactoring
on its own may seem like it is making it more complicated, but when you realize
you are moving all of this code out of a bloated class it makes more sense.

```ruby 4-service-objects https://github.com/tute/refactoring-workshop/blob/master/4-service-objects/app.rb Full Source Code

class User
  attr_accessor :subscription

  # validations
  # callbacks
  # authentication logic
  # notifications logic

  def subscribe
    api_result = try_api { PaymentGateway.subscribe }
    if api_result == :success
      self.subscription = :monthly_plan
    end
    api_result
  end

  def unsubscribe
    api_result = try_api { PaymentGateway.unsubscribe }
    if api_result == :success
      self.subscription = nil
    end
    api_result
  end

  private

  # Try API connection, trap and log it on failures
  def try_api
    yield
  rescue SystemCallError => e
    # log API connection failure
    :network_error
  end

  # Other private methods
end

```

Now all of this code really should be part of some subscription class, but since it
depends on the user it seems to have just been crammed into this code.  It doesn't
look like much, but remember this is a user model that would have hundred of other
lines of code, likely with more examples of logic that needs to be moved. To move this
code out though you are going to need pass along the user. Originally I wanted to make
a instance variable called user and pass him into the initialization, but I found out
that there is a slightly sexier way to go about it through the use of a struct.

```ruby Tute Costa's Solution https://github.com/tute/refactoring-workshop/blob/tute-solutions/4-service-objects/app-solution-tute.rb Full Source Code

class SubscriptionService < Struct.new(:user)
  def subscribe
    api_result = try_api { PaymentGateway.subscribe }
    if api_result == :success
      user.subscription = :monthly_plan
    end
    api_result
  end

  def unsubscribe
    api_result = try_api { PaymentGateway.unsubscribe }
    if api_result == :success
      user.subscription = nil
    end
    api_result
  end

  private

  # Try API connection, trap and log it on failures
  def try_api
    yield
  rescue SystemCallError => e
    # log API connection failure
    :network_error
  end
end

class User
  attr_accessor :subscription


  # Podría ser delegate :subscribe, :unsubscribe, to: :subscription_service

  def subscribe
    subscription_service.subscribe
  end

  def unsubscribe
    subscription_service.unsubscribe
  end

  private

  def subscription_service
    @subscription_service ||= SubscriptionService.new(self)
  end
end

```

This allows us to simply pass the user into `SubscriptionService.new(self)` and
from there we can call user instead of self. This really cleans up the user model.
Also we make sure to call subscription_service were a new instance is created only
if it is needed.

##Summary

In summary the biggest favor you can do yourself before refactoring is to make sure
you have a comprehensive test suit. Once that is in place you are able to freely change
code without having to worry about secretly breaking functionality. To give this a try
for yourself visit fork the repository here `https://github.com/tute/refactoring-workshop`,
and refactor away. Run the tests suit to ensure you didn't break anything. Good luck, and happy
coding.
