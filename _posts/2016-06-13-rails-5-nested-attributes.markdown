---
title:  "Rails 5 API-only and Nested Attributes"
date:   2016-06-13 21:36:20 +0000
layout: post
image:  '/images/rails.webp'
tags: [Programming]
---

One of the new features bundled with Rails 5 is the ability to build an API-only Rails site. That is, a Rails site with no views or assets or turbolinks, or indeed anything else related to browser presentation. It’s a feature that was previously available through the use of the rails-api gem, which has now made its way into the Rails core. This is in part a nod towards the rise of microservices; services which, in their simplest form, will dish out some JSON, or XML, or protobuf, or whatever, to be consumed by another.

A useful approach is to then slap a Javascript framework (such as Angular, or Backbone) in front of the API, to render the information in a human-friendly manner. This not only allows developers to utilise the powerful browser-bending capabilities of these frameworks, but also nicely de-couples them from the backend.

Recently I have found myself doing just that - building an Angular app to poke and prod a backend API written in Rails. Sounds easy. But before long I had reached an impasse where Angular would post some JSON to Rails, only for Rails to point-blank reject to process it. My problems, it transpired, were centred around the fickle *accept_nested_attributes_for* directive and I record them here to hopefully save others from the certain amount of head-scratching I encountered.

Quick recap: *accepts_nested_attributes_for* is an ActiveRecord directive that allows one model to create or update instances of other related models, nested beneath its own. It is most often used in large forms that require input for multiple types of information (for instance, address details in a customer information form might be stored as an Address model).

The particular API call which led to my impasse was the creation of an Event, which is made up of a number of Tasks. Here are the model definitions as they ended up:

```ruby
class Event < ApplicationRecord
  has_many :tasks
  accepts_nested_attributes_for :tasks, :allow_destroy => true
end

class Task < ApplicationRecord
  belongs_to :event, optional: true
end
```

The Event model defines a *has_many* relationship with Tasks, and states that it will accept nested attributes to create those tasks. It also allows the deletion of nested Tasks, through the `:allow_destroy` option.

The Task model defines a *belongs_to* relationship with an Event, as you might expect; but it also defines the presence of that Event as optional. This is important. In Rails 5, [belongs_to relationships are now required by default][rails-pr]. Without this attribute, the model validation will fail with “Event must exist” when it tries to create Tasks for which an Event does not yet exist.

The *accepts_nested_attributes_for* method actually causes an ‘attribute writer’ to be defined on the parent model. This is a method which by convention is named after the nested model, with the postfix `_attributes`. In our case this becomes `tasks_attributes`.

This can be seen in the following example of a Post request that will create an Event with 2 nested Tasks:

```json
{
   "event" : {
      "name" : "My Smashing Event",
      "event_type" : "ReallyBusyEvent",
      "start_time" : "25 May 12:56",
      "end_time" : "25 May 13:56"
      "tasks_attributes" : [
         {
            "action" : "do_something",
            "start_time" : "25 May 12:30"
         },
         {
            "start_time" : "25 May 12:40",
            "action" : "do_something_else"
         }
      ],
   }
}
```
Except it won’t. Because first we have to bribe the doorman, Strong Parameters. Without this step, the request will simply be rejected off the bat when it fails the initial security screening.

In the `app/controllers/events_controller.rb` file, we have:

```ruby
def event_params
  params.require(:event).permit(:name, :event_type, :start_time, :end_time, tasks_attributes: [  :id, :start_time, :action ] )
end
```

Now the syntax here is somewhat counter-intuitive: it seems to suggest we expect a single array of `:id, :start_time, :action`. But in actuality it will expect an array of _objects_ (or hashes, once Rubyised) with keys `:start_time` and `:action`.

Note the inclusion of the `:id` parameter too (and its absence in our JSON above), without this you'll find that any _updates_ result in new nested attributes being created instead of amending those that already exist. 

And that’s it. With these formalities dispensed, you can start crafting JSON requests to create nested attributes against your Rails 5 API-only site.

[rails-pr]: https://github.com/rails/rails/pull/18937

