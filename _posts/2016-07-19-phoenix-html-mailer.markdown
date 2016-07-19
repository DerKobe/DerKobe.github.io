---
layout: post
title: "Phoenix HTML Mailer"
date: 2016-07-19T09:55:25+02:00
---

Sooner or later you want to send (HTML) emails with Phoenix. Rails has Mailers for this and Phoenix has a 
[guide](http://www.phoenixframework.org/docs/sending-email) that describes how to send basic emails.

I wanted a system that you can use like a Controller or Model the _Phoenix way_, so I wrote a bunch of modules 
that let's you easily send emails.

The actual sending of the emails will be done via [Mailgun](https://www.mailgun.com/).
I find Mailgun to be an excellent service and even the free plan will satisfy the requirements of most people.
 
Since I'm not a fan of long prose let's dig right into code:

# lib/my_app/mailer.ex
{% highlight elixir %}
defmodule MyApp.Mailer do
  def mailer do
    quote do
      @from "info@myapp.com"

      def perform(mail), do: send_directly(mail, true)

      defp send_directly(mail, convert \\ false) do
        mail = if convert do
          mail
          |> enforce_atom_keys
          |> (fn map -> Map.put(map, :params, (enforce_atom_keys(map.params) |> Map.to_list)) end).()
          |> Map.to_list
        else
          mail
        end

        send_email(
          to:      mail[:to],
          from:    mail[:from],
          subject: mail[:subject],
          html:    Phoenix.View.render_to_string(MyApp.EmailView, mail[:template], mail[:params])
        )
        Logger.debug "Email sent to #{mail[:to]} with subject '#{mail[:subject]}'"
      end

      defp send_async(mail) when is_list(mail), do: send_async(Enum.into(mail, %{}))
      defp send_async(mail) do
        case Exq.enqueue(Exq, "mailer", __MODULE__, [mail]) do
          {:ok, ack} -> Logger.debug("Successfully enqueued email to #{mail[:to]} with subject #{mail[:subject]}")
          error      -> Logger.error("Failed to enqueue mail to #{mail[:to]} with subject #{mail[:subject]}")
        end
      end

      defp enforce_atom_keys(map), do: for {key, val} <- map, into: %{}, do: {String.to_atom(key), val}

    end
  end

  defmacro __using__(_) do
    apply(__MODULE__, :mailer, [])
  end
end
{% endhighlight %}

This module is the core of the actual mailing process. Since you want to be able to send your mails asynchronously with retries when failing,
I added the [Exq](https://github.com/akira/exq) job queue system. Exq uses Redis for queueing (like Ruby's Resque) under the hood, so you need to have a Redis system available.

The `enforce_atom_keys` function ensures that the passed parameters look the same regardless of using the direct or asynchronous sending method
(if async then the params get serialized and the keys will be strings).

Every Mailer doubles as an Exq Worker (hence the `perform` function). So when the module gets async-executed by Exq, the perform function will just call the direct sending mechanism.

The `defp send_async(mail) when is_list(mail)` is needed because the parameters can be passed by Map or by List (which again is due to serialization by Exq).


# web/web.ex
{% highlight elixir %}
defmodule MyApp.Web do
  
  ...
  
  def mailer do
    quote do
      use MyApp.Mailer
      use Mailgun.Client, domain: System.get_env("MAILGUN_API_ENDPOINT"), key: System.get_env("MAILGUN_API_KEY")
    end
  end
  
  ...
  
end
{% endhighlight %}  


# web/mailer/example_mailer.ex
{% highlight elixir %}
defmodule MyApp.ExampleMailer do
  use MyApp.Web, :mailer

  def send(email, name) do
    send_async(
      to:       email,
      from:     @from,
      subject:  "Example Mailer Subject",
      template: "example.html",
      params:   %{name: name}
    )
  end
end
{% endhighlight %}

You can decide per Mailer if you want to send the mail directly or asynchronously but in almost all cases async is the way to go.


# web/views/email_view.ex
{% highlight elixir %}
defmodule MyApp.EmailView do
  use MyApp.Web, :view
end
{% endhighlight %}

The View is straight forward. Templates will be placed in `web/templates/email/` as regular `.html.eex` template files.
 
# web/templates/email/example.html.eex
{% highlight erb %}
<h1>Welcome <%= @name %>!</h1>
<p>This is an HTML email example ... apparently.</p>
{% endhighlight %}
 
 
Now you can use the Mailer anywhere to send emails like so `MyApp.ExampleMailer.send("phil@jack.chat", "Philip")`

That's about it.

I always found _designing_ and _building_ HTML emails one of the harder things to do when dealing with this topic.
But recently I discovered [Mjml](https://mjml.io) which is a very nice tool if you want to keep your sanity while working with HTML email designs.
