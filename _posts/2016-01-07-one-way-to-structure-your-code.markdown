---
layout: post
title: "One Way to Structure Your Code"
date: 2016-01-07T09:55:40+01:00
---

I worked with different project sizes (in terms of LoC) along my way. And I have seen a lot of projects with either bloated controllers or models ... or both.
This is widely known as fat x skinny y. But how about skinny x skinny y?
The method you're about to read about how to structure code to avoid this, was not entirely mine. I read about something similar somewhere long ago and loved it right away. 
But sadly I can't remember where I read it. Even sadder I have seen it IRL way to rarely. But maybe this is just my opinion, so judge for yourself.

The idea is to move code out of controllers and models that does not belong there measured by very strict and almost inquisitiony standards.
In models just keep the really-general-every-instance-of-this-must-have stuff. In controllers only "controlling" code is allowed.

A few examples:

- There's a method in your Ruby model to create an admin user? That's a paddlin'.
- In a controller there is a block of code getting records from the Database via a model and mapping the data to something to pass to the template? That's a paddlin'.
- There're callbacks in a model? You better believe that's a paddlin'.

<center><img src="/assets/paddlin.png" width="100" height="100" style="border-radius: 5px;" /></center>
 
 
But where do I put the code?

# Services

A service is just a Ruby module with a bunch of methods. I used services whenever I did not need to store state along the way of computing the outcome.
For example I had this code to get a consecutive list of successful and aborted registrations. 
The code does not belong in the `users` model because it's purpose is too specific.
 
{% highlight ruby %}
module Services::Registrations
  class << self
    def successful
      make_sequential_series(
        query <<-SQL.strip_heredoc
          SELECT to_char(created_at, 'YYYY-MM-DD') AS created_at, COUNT(*) AS cnt
          FROM users
          WHERE contact IS NOT NULL
          GROUP BY to_char(created_at, 'YYYY-MM-DD')
          ORDER BY created_at
        SQL
      )
    end

    def aborted
      make_sequential_series(
        query <<-SQL.strip_heredoc
          SELECT to_char(created_at, 'YYYY-MM-DD') AS created_at, COUNT(*) AS cnt
          FROM users
          WHERE contact IS NULL AND redirect_id IS NULL
          GROUP BY to_char(created_at, 'YYYY-MM-DD')
          ORDER BY created_at
        SQL
      )
    end

    def query(sql)
      ActiveRecord::Base.connection.execute(sql)
    end

    def make_sequential_series(data)
      compact_stats = data.reduce({}) do |data, item|
        data[Date.parse(item['created_at']).to_s] = Integer(item['cnt'])
        data
      end

      (30.days.ago..Date.today).map do |date|
        { d: date, c: compact_stats[date.to_s].present? ? compact_stats[date.to_s] : 0 }
      end
    end
  end
end
{% endhighlight %}

If the code was in the model the call would still look very much the same (`User.successful_registration` vs. `Services::Registrations.successful`). 
But the code and tests are easier to read and maintain (for both the user-model and the registrations-service).

# Processes

I use services almost always to compute stuff with no side effects. Processes on the other hand are almost always more complex and involve more steps along the way. Their main purpose is the side effects and not the return value.
  
{% highlight ruby %}
module MessageProcesses
  class Create
    attr_reader :user, :channel, :text, :type, :message, :link

    MAX_TEXT_LENGTH = 500

    def initialize(user:, channel:, text:, type:)
      @user    = user
      @channel = channel
      @text    = text
      @type    = type
      @origin  = origin
      @link    = nil
    end

    def run
      if type == 'chat'
        remove_markdown_images
        substitute_giphy
        truncate_text
        get_embed_for_first_link
      else
        generate_announcement_text(type)
      end

      Message.create(user: user, channel: channel, text: text, type_: type, link: link) unless text.empty?

      self
    end

    private
    # ... implementation for remove_markdown_images, substitute_giphy, truncate_text, get_embed_for_first_link, generate_announcement_text
  end
end
{% endhighlight %}

This particular process was used (very much) like this in a controller: 
{% highlight ruby %}
def create
  process = MessageProcesses::Create.new(
    user:    User.find_by(auth_token: params[:auth_token]),
    channel: params[:channel],
    text:    params[:text],
    type:    params[:type]
  ).run

  render json: process.message
end
{% endhighlight %}

The benefit here is that all the code in order to transform the input data and create the message is encapsulated in a class. Again the code and tests are easier to read and maintain (for both the message-model and the message-create-process).

Of course processes can call services or other processes or the other way around. 

# Conclusion

So far this concept worked out great for me and the teams I worked with. By having to give services and processes names you start to think more modular and concepts get more visible instead of one gooey flow of data through controllers, models, and views.

---

# Comments
_If you want to comment on, or talk about this post you can do this via [<img src="/assets/talk-about-jack.png" width="32" height="32" title="Talk About Jack" />](http://jack.chat) (just [install the Chrome Extension](https://chrome.google.com/webstore/detail/talk-about-jack/mfjhkijmchogjenmblohgkifnakapbhf) and leave a message in the "Page" channel of this post).
And of course you're always welcome to reach out to me via the "Domain" Channel._
