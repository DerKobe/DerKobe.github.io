---
layout: post
title: "Email Tracking Pixel in Elixir Phoenix"
date: 2016-12-24T09:34:32+01:00
---

I needed to track wether or not an user opened a notification email sent by my app (to prevent spamming him with further emails).
So I embedded an img-Tag in the email pointing to something like https://myhost.com/:identifier.gif.

Now I had to find out about two things:

1. The smallest possible gif there is
2. How to serve an image from a Phoenix controller

For the first one I found [this Github repo by mathiasbynens](https://github.com/mathiasbynens/small) with a huge list of all kinds of file types and their smallest possible valid representation.

I downloaded the gif from the repo, opened an IEx shell and loaded the file. Then I printed the bitstring representation to the console and copy&pasted it into my controller.

For the second part I just googled my problem and found a simple example on stackoverflow.

Then I put it all together and this is the result:

{% highlight elixir %}
defmodule MyApp.TrackingController do
  use MyApp.Web, :controller

  @minigif <<71,73,70,56,57,97,1,0,1,0,128,0,0,255,255,255,0,0,0,33,249,4,1,0,0,0,0,44,0,0,0,0,1,0,1,0,0,2,2,68,1,0,59>>

  def track_email(conn, %{"identifier" => identifier}) do
    identifier = identifier |> String.split(".") |> List.first

    # do something with this information

    conn
    |> put_resp_content_type("image/gif")
    |> send_resp(200, @minigif)
  end
end
{% endhighlight %}

And in the email template I put something like this at the end:

{% highlight elixir %}
<img src="<%= ~s(#{@host}/api/track-email/#{@identifier}.gif) %>" />
{% endhighlight %}

BTW on how to more comfortable send emails in Elixir Phoenix I wrote [a post about it here](/2016/07/19/phoenix-html-mailer.html).
