---
layout: post
title: "Get Notified When An User Leaves A Phoenix Channel"
date: 2016-01-11T19:44:16+01:00
---

This post is based on the stackoverflow answer by [Chris McCord](https://twitter.com/chris_mccord) to my question on this topic ([see here](http://stackoverflow.com/questions/33934029/how-to-detect-if-a-user-left-a-phoenix-channel-due-to-a-network-disconnect)).
 
I'm working on a project where I wanted to trigger a function when an user leaves a Phoenix channel due to closing the app, losing network connection and so on (basically every "leaving event" where the app is not able to notify the server).
The answer is using a [GenServer](http://elixir-lang.org/docs/v1.2/elixir/GenServer.html) to watch the established connection and trap the exit.  

# lib/my_app.ex
{% highlight elixir %}
children = [
  ...
  worker(ChannelWatcher, [:rooms])
]
{% endhighlight %}

# web/channels/user_socket.ex
{% highlight elixir %}
def connect(%{"signed_token" => signed_token, "origin" => origin}, socket) do
   # the signed_token is passed by the client which got it prior 
   # to connecting to the channel from some init call
  case Phoenix.Token.verify(socket, "user_id", signed_token) do
    {:ok, user_id} ->
      socket = assign(socket, :user_id, user_id)
      {:ok, socket}
    {:error, _} -> 
      :error
    end
  end
end
{% endhighlight %}

# web/channels/room_channel.ex
{% highlight elixir %}
def join("rooms:", <> room_id, params, socket) do
  user_id = socket.assigns.user_id
  data = [room_id, user_id] # pass any data you want to access 
                            # when the connection is closed
  :ok = ChannelWatcher.monitor(:rooms, self(), {__MODULE__, :leave, data})
  {:ok, socket}
end

def leave(room_id, user_id) do
  # Collect the grand prize!
end
{% endhighlight %}

# lib/my_app/channel_watcher.ex
{% highlight elixir %}
defmodule ChannelWatcher do
  use GenServer

  ## Client API

  def monitor(server_name, pid, mfa) do
    GenServer.call(server_name, {:monitor, pid, mfa})
  end

  def demonitor(server_name, pid) do
    GenServer.call(server_name, {:demonitor, pid})
  end

  ## Server API

  def start_link(name) do
    GenServer.start_link(__MODULE__, [], name: name)
  end

  def init(_) do
    Process.flag(:trap_exit, true)
    {:ok, %{channels: HashDict.new()}}
  end

  def handle_call({:monitor, pid, mfa}, _from, state) do
    Process.link(pid)
    {:reply, :ok, put_channel(state, pid, mfa)
  end

  def handle_call({:demonitor, pid}, _from, state) do
    case HashDict.fetch(state.channels, pid) do
      :error -> 
        {:reply, :ok, state}
      {:ok,  _mfa} ->
        Process.unlink(pid)
        {:reply, :ok, drop_channel(state, pid)}
    end
  end

  def handle_info({:EXIT, pid, _reason}, state) do
    case HashDict.fetch(state.channels, pid) do
      :error -> {:noreply, state}
      {:ok, {mod, func, args}} ->
        Task.start_link(fn -> apply(mod, func, args) end)
        {:noreply, drop_channel(state, pid)}
    end
  end

  defp drop_channel(state, pid) do
    %{state | channels: HashDict.delete(state.channels, pid)}}
  end

  defp put_channel(state, pid, mfa) do
    %{state | channels: HashDict.put(channels, pid, mfa)}}
  end
end
{% endhighlight %}

# Conclusion
This system works very well for me in production. And compared to Faye with which I worked in the past, I begin to see the true power of Elixir/Erlang. It's just a blast!

---

# Comments
_If you want to comment on, or talk about this post you can do this via [<img src="/assets/talk-about-jack.png" width="32" height="32" title="Talk About Jack" />](http://jack.chat) (just [install the Chrome Extension](https://chrome.google.com/webstore/detail/talk-about-jack/mfjhkijmchogjenmblohgkifnakapbhf) and leave a message in the "Page" channel of this post).
And of course you're always welcome to reach out to me via the "Domain" Channel._
