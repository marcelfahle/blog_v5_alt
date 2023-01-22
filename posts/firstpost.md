---
title: Video Uploads with Phoenix LiveView and Mux
description: A complete walkthrough on how to upload video files to Mux using Phoenix LiveView
date: 2022-12-17
tags:
  - video
  - mux
  - phoenix
  - liveview
  - elixir
layout: layouts/post.njk
---
Uploading large video files is something that I often do when I work on Bold. But even if you rely on an incredible API like Mux (which we do) or use all the shortcuts that Phoenix and LiveView provide, getting everything set up correctly is not always trivial. 

So let's walk through a complete example using Phoenix LiveView and Mux from start to finish.

## Setup
We start by creating a new phoenix app (we're using Phoenix 1.7-rc in this example):

```shell
mix phx.new watermarkr --live --no-dashboard --binary-id && cd watermarkr
```

Note: We're using `binary_id` as our primary key type because it makes it later easier to tie our own video records to the assets from Mux.

Next, we'll add a video resource and use the Phoenix generators to keep it simple. For now, we'll have a title and the mux asset and playback IDs to access our content and later play even it:

```shell
mix phx.gen.live Media Video videos title:string asset_id:string playback_id:string
```

Let's grab the routes from the prompt and add them to our router:

```elixir
# lib/watermarkr_web/router.ex

scope "/", WatermarkrWeb do
    pipe_through :browser

    get "/", PageController, :home

    live "/videos", VideoLive.Index, :index
    live "/videos/new", VideoLive.Index, :new
    live "/videos/:id/edit", VideoLive.Index, :edit

    live "/videos/:id", VideoLive.Show, :show
    live "/videos/:id/show/edit", VideoLive.Show, :edit
end
```

We need to create our Video IDs manually because we already need one before saving our video resource, so we can tie it to the Mux Upload.

So, let's also turn off the autogeneration of IDs on the database level and, instead, make sure we're able to add one manually by adding it to the `cast` and `validate_required` function. And while we're here, we can also quickly remove the asset and playback_ids from our changeset validation so that those won't trouble us later during our client-side form validation (we'll set those via Mux webhook later):

```elixir
# lib/watermark/media/video.ex

defmodule Watermarkr.Media.Video do
  use Ecto.Schema
  import Ecto.Changeset

  @primary_key {:id, :binary_id, autogenerate: false}
  @foreign_key_type :binary_id

  schema "videos" do
    field :asset_id, :string
    field :playback_id, :string
    field :title, :string

    timestamps()
  end

  @doc false
  def changeset(video, attrs) do
    video
    |> cast(attrs, [:id, :title, :asset_id, :playback_id])
    |> validate_required([:id, :title])
  end
end
```

Now it's time to create our ID. In our LiveView, let's generate one as soon as the User hits the "New" button:

```elixir
# lib/watermarkr_web/live/video_live/index.ex

  defp apply_action(socket, :new, _params) do
    socket
    |> assign(:page_title, "New Video")
    |> assign(:video, %Video{id: Ecto.UUID.generate()})
  end
```

We must also ensure that our manually generated ID gets passed down to the `Media.create_video/1` function. We can add it to the video_params inside the `save_video/3` function:

```diff-elixir
# lib/watermarkr_web/live/video_live/index.ex

defp save_video(socket, :new, video_params) do
+  video_params = Map.put(video_params, "id", socket.assigns.video.id)

  case Media.create_video(video_params) do 
[..]
```

Afterward, we create our database and run our migration (make sure your database connection is configured correctly in `config/dev.exs`):

```shell
mix ecto.create && ecto.migrate
```

Once you start the dev server with `mix phx.server` and navigate to [http://localhost:4000/videos](http://localhost:4000/videos), you'll see that we now have a mighty fine UI available for our video resource (Thanks, Team Tailwind!)

![Our first app](/img/videoupload-screen1.png)

## File upload

Let's tackle the video uploads next. 
We start by adding the Mux API Wrapper to our dependencies.

```diff-elixir
# mix.exs
defp deps do
  [
    # [...]
+    {:mux, "~> 2.5.0"}
  ]
end
```

Now add the Mux Access Token ID and Secret Key, which you can grab from your Mux settings page, to our config/dev.exs and run mix deps.get afterward:

```diff-elixir
# config/dev.exs

+config :mux,
+  access_token_id: "MUX_TOKEN_ID",
+  access_token_secret: "MUX_TOKEN_SECRET"
```

The way these kinds of uploads work is basically, once a user initiates an upload, we're requesting a signed upload URL from Mux, which is unique to that current upload. This URL is then handed over to the JavaScript client (we use UpChunk), which takes care of the actual uploading process for us.

We'll plug the uploader directly into our create video modal so that the actual upload only happens once the User clicks "Save Video" and submits the form.

Let's start by accepting uploads for our video resource and let our LiveView request the signed upload URL. To do that, we need a few things: 

First, we'll configure our uploader inside our video form's [update/2](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveComponent.html#c:update/2) callback using [Phoenix.LiveView.allow_upload/3](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#allow_upload/3). We'll pass it a presign_upload/2 function to request our (external) signed URL from Mux. We'll write that function in a little bit:

```elixir
# lib/watermarkr_web/live/video_live/form_component.ex
  @impl true
  def update(%{video: video} = assigns, socket) do
    changeset = Media.change_video(video)

    {:ok,
      socket
      |> assign(assigns)
      |> allow_upload(:video_file, 
        accept: :any,
        max_file_size: 100_000_000, 
        external: &presign_upload/2
      )
      |> assign(:changeset, changeset)}
  end
```

This will give us an uploads assign on our socket connection, which lets us render our file input field. Let's add that to our video form now:

```html
{% raw %}
# lib/watermarkr_web/live/video_live/form_component.ex
# [...]
    <.input field={{f, :title}} type="text" label="title" />
    <.input field={{f, :has_watermark}} type="text" label="has_watermark" />
    <.live_file_input upload={@uploads.video_file} />
{% endraw %}
```

![Upload Field](/img/videoupload-screen2.png)

Once we hit the submit button, LiveView will process all the configured uploads in our form before invoking the `handle_event/3` callback for submitting the form. 

That means we need to take care of the actual uploading next. We will use the `presign_upload/2` function to request a signed URL from Mux to begin uploading. We will also add our own video ID, which we created earlier, as a passthrough value, so we can link the Mux asset to our video resource once encoding is complete.

```elixir
# lib/watermarkr_web/live/video_live/form_component.ex

  defp presign_upload(_entry, socket) do
    client = Mux.client()
    video = socket.assigns.video

    params = %{
      "new_asset_settings" => %{
        "passthrough" => video.id,
        "playback_policies" => ["public"]
      },
      "cors_origin" => "*"
    }

    {:ok, %{"url" => url}, _client} = Mux.Video.Uploads.create(client, params)

    {:ok, %{uploader: "UpChunk", endpoint: url}, socket}
  end
```

Signed URL in hand, we'll stuff that together with the information which uploader to use into a map (`%{uploader: "UpChunk", endpoint: url}`), which LiveView will then hand over to JavaScript. Let's tackle that part next:
First, we need to add UpChunk to our JS dependencies:

```shell
npm install --prefix assets --save @mux/UpChunk
```

Next, we'll add the uploader function to our JS: 

```js
// assets/js/app.js
import * as UpChunk from "@mux/upchunk";

export let uploaders = {};

uploaders.UpChunk = function (entries, onViewError) {
  // create upchunk upload with signed url (endpoint)
  // and file object received from liveview
  entries.forEach((entry) => {
    let {
      file,
      meta: { endpoint },
    } = entry;
    let upload = UpChunk.createUpload({ endpoint, file });
    // stop upload on error and report back
    // to liveview
    onViewError(() => upload.pause());
    upload.on("error", (e) => entry.error(e.detail.message));

    // report progress and success back to liveview
    upload.on("progress", (e) => {
      if (e.detail < 100) {
        entry.progress(e.detail);
      }
    });
    upload.on("success", () => entry.progress(100));
  });
};

let liveSocket = new LiveSocket("/live", Socket, {
  uploaders, 
  params: {_csrf_token: csrfToken}
})
```

## Upload Progress 

In the code above, you'll see a few functions being called on the `entry` object (`progress()`, `error()`). Those are the callbacks our LiveView form provides to communicate back to it. Let's use that information to show the upload progress. We'll do that by adding a little progress bar to our markup, and I've added mine right under the [live_file_input](https://hexdocs.pm/phoenix_live_view/Phoenix.Component.html#live_file_input/1) component:

```html
// lib/watermarkr_web/live/video_live/form_component.ex

<div :for={entry <- @uploads.video_file.entries} class="w-full bg-gray-200 rounded-full h-2.5">
  <div class="bg-blue-600 h-2.5 rounded-full" style={"width: #{entry.progress}%"}></div>
</div>
```

Easy peasy.

## Webhooks

The last missing piece to the puzzle is to wait for the Mux encoder to do its job processing our uploads. Thankfully, Mux will notify us via webhooks when everything is done, and all we need to do is to wait and listen for those webhooks. 

We'll start by adding an API route to our router, which Mux can then trigger with status updates:

```elixir
# lib/watermarkr_web/router.ex

scope "/api", WatermarkrWeb do
  pipe_through :api

  post "/webhooks/mux", WebhookController, :mux
end
```

### Webhooks on localhost:

You're most likely developing this on a local development machine, a.k.a. your localhost, which is impossible for Mux to notify directly. 

The easiest way to make your localhost available to Mux and the outside world is by providing a secure tunnel using a tool like ngrok. It is a free tool and works great for our purpose. 

So, if your Phoenix app runs on localhost:4000, all you need to do is run ngrok with that port number, like so:

```shell
ngrok http 4000
```

![ngrok](/img/videoupload-screen3-ngrok.png)

You'll get back a URL, which we can then add to our Mux dashboard under Settings -> Webhooks -> Create new Webhook.

![Create a Webhook in Mux](/img/videoupload-screen4-webhook.png)

Once we've created a webhook, we still need to grab the signing secret to verify that all incoming webhooks are actually from Mux and are legit. So, on the webhooks page, click on the "Show Signing Secret" button of your newly created webhook and add that secret to your Mux config:

![Show Webhook Signing Secret in Mux](/img/videoupload-screen5-webhooksecret.png)

```diff-elixir
# config/dev.exs

config :mux,
  access_token_id: "MUX_TOKEN_ID",
  access_token_secret: "MUX_TOKEN_SECRET",
+  webhook_secret: "WEBHOOK_SIGNING_SECRET"
```

Next, we'll set up a controller to process those webhooks. We'll start with a very simple version that logs every incoming webhook to our terminal and responds with a success message:

```elixir
# lib/watermarkr_web/webhook_controller.ex

defmodule WatermarkrWeb.WebhookController do
  use WatermarkrWeb, :controller

  def mux(conn, params) do
    IO.inspect(params)

    json(conn, %{message: "webhook received"})
  end
end
```

If we now upload a video, we should see the notifications from Mux pouring into your server console:

```json
%{
  "accessor" => nil,
  "accessor_source" => nil,
  "attempts" => [],
  "created_at" => "2023-01-21T02:58:10.905000Z",
  "data" => %{
    "aspect_ratio" => "16:9",
    "created_at" => 1674269889,
    "duration" => 37.1811,
    "id" => "7zFS02acd3buTmutRrC78yvgPVgcLF4hG",
    "master_access" => "none",
    "max_stored_frame_rate" => 29.97,
    "max_stored_resolution" => "HD",
    "mp4_support" => "none",
    "passthrough" => "8dd4676b-579d-4bc7-8896-3fe6638decab",
    "playback_ids" => [
      %{"id" => "nz6AXjfZUjOT0scdnbGB1IuqHkzj9CQtQ", "policy" => "public"}
    ],
    "status" => "ready",
    "tracks" => [
      %{
        "duration" => 37.137104,
        "id" => "WFUxs018nVTeog6fczNEWofPBp4N00asB",
        "max_channel_layout" => "stereo",
        "max_channels" => 2,
        "type" => "audio"
      },
      %{
        "duration" => 37.1371,
        "id" => "yrwviIQ4tcadfRmIcNa3fLGhC00I01JmQxMdV9rs9hd100",
        "max_frame_rate" => 29.97,
        "max_height" => 1080,
        "max_width" => 1920,
        "type" => "video"
      }
    ],
    "upload_id" => "UvkzTWnemeMESr0002Z6mkvibNPuVtlfu4g"
  },
  "environment" => %{"id" => "4p44dv", "name" => "Development"},
  "id" => "9133be58-0797-4d0c-8123-2a302bd9ee9d",
  "object" => %{"id" => "7zFS0edgY3buTmutRrC78yvgPVgcLF4hG", "type" => "asset"},
  "request_id" => nil,
  "type" => "video.asset.ready"
}
```

There's a lot of information in those payloads, but the most interesting parts for us right now are

- `type`: There are many types of webhook events (complete list of webhook events), but we're currently only interested in "video.asset.ready", which tells us that the video has been encoded successfully and is ready for playback.
- `data.id`: The ID of the Mux asset, which we can use to interact and request information about the video we've just uploaded
- `data.playback_id[].id`: The Playback ID, which we can use to actually play our video
- `data.passthrough`: our own video ID, which we added to the upload earlier.

So, let's now parse the webhook payloads, grab the above information, and update our video in our database.

It's always a good idea to verify incoming webhook requests to make sure it's actually Mux sending us that data. But to do any kind of verification, we need the raw payload that Mux is sending us, which we don't have anymore at this point. The data already went through Phoenix's Plug pipeline and has been parsed into a map, which is usually much more useful. But in our case, we need the raw JSON data, and the trick is to stick a copy of it onto our connection before it gets turned into a Map. We'll write a little helper script to do just that (I got that solution from [GitHub](https://github.com/phoenixframework/phoenix/issues/459#issuecomment-440820663) and [Stack Overflow](https://stackoverflow.com/a/51587646/1136313)):

```elixir
# lib/watermarkr/body_reader.ex

defmodule Watermarkr.BodyReader do
  def read_body(conn, opts) do
    {:ok, body, conn} = Plug.Conn.read_body(conn, opts)
    conn = update_in(conn.assigns[:raw_body], &[body | &1 || []])
    {:ok, body, conn}
  end
end
```

Now we can provide this function to the `:body_reader"` option of the Plug.Parsers behavior:

```diff-elixir
# lib/watermarkr_web/endpoint.ex

  plug Plug.Parsers,
    parsers: [:urlencoded, :multipart, :json],
    pass: ["*/*"],
+    body_reader: {Watermarkr.BodyReader, :read_body, []},
    json_decoder: Phoenix.json_library()

```

The Mux SDK comes with a [Mux.Webhooks.verify_header/4](https://hexdocs.pm/mux/Mux.Webhooks.html#verify_header/4) function, which makes the last verification step a breeze. We'll give it our webhook signing secret, which we set up earlier, as well as the webhook's signature header and the raw JSON that we now have access to: 

```elixir
# lib/watermarkr_web/webhook_controller.ex

defmodule WatermarkrWeb.WebhookController do
  use WatermarkrWeb, :controller

  alias Watermarkr.Media

  def mux(conn, params) do
    # grab the mux signature and full payload for verification
    signature_header = List.first(get_req_header(conn, "mux-signature"))
    raw_body = List.first(conn.assigns.raw_body)

    # verify that the incoming webhook is legit and only then 
    # process is. If verfication fails, return an error.
    case Mux.Webhooks.verify_header(
         raw_body,
         signature_header,
         Application.fetch_env!(:mux, :webhook_secret)
       ) do

      :ok ->
        # process the webhook payload
        process_mux(params["type"], params["data"])

        # respond to the webhook that it has been processed
        conn
        |> put_resp_content_type("application/json")
        |> send_resp(200, "")

      {:error, message} ->
        conn
        |> put_status(400)
        |> json(%{message: message})
    end
  end

  # we pattern match on the webhook type, that way
  # it's easy to add more types to process
  defp process_mux("video.asset.ready", %{"id" => asset_id, "passthrough" => video_id, "playback_ids" => playback_ids}) do
    # find our video in the database using the 
    # passthrough value and updated it with the 
    # asset and playback ids.
    # A Mux asset can have multiple playback ids but
    # in our example we always grab the first one. 
    video = Media.get_video!(video_id) 
    playback_id = hd(playback_ids)["id"]
    Media.update_video(video, %{asset_id: asset_id, playback_id: playback_id})
    
  end

  # ignore all other webhook events
  defp process_mux(_type, _asset), do: :ok

end
```

## Bonus: Live updates 

Cool, we're now updating our database after receiving webhooks from Mux. But waiting for those webhooks and manually refreshing our browser to see the result is not very LiveView'y. Let's update our LiveView once our video has been updated. 

We'll use [Phoenix.PubSub](https://hexdocs.pm/phoenix_pubsub/Phoenix.PubSub.html) to notify our LiveView about the updates. Everything is already set up on Phoenix's end, so all we need to do is to broadcast our message. I tend to call my broadcast functions from within context modules, especially regarding any sort of CRUD action, so let's do this here as well. We'll write a private `notify/2` function that we can then stick into our pipelines:

```diff-elixir
# lib/watermarkr/media.ex

  def update_video(%Video{} = video, attrs) do
    video
    |> Video.changeset(attrs)
    |> Repo.update()
+    |> notify({:video_updated, video.id})
  end

+  defp notify(video, message) do
+    Phoenix.PubSub.broadcast(Watermarkr.PubSub, "video_updates", message)
+    video
+  end

```

Now, back in our Index LiveView, we first need to subscribe to the topic of our broadcasts (`video_updates` in our example):

```elixir
# lib/watermarkr_web/live/video_live/index.ex

[...]
  def mount(_params, _session, socket) do
    if connected?(socket) do
      Phoenix.PubSub.subscribe(Watermarkr.PubSub, "video_updates")
    end
[...]
```

And then, further down, we'll add a [handle_info/2](https://hexdocs.pm/phoenix_live_view/Phoenix.LiveView.html#c:handle_info/2) callback and refetch all our videos if there has been an update:

```elixir
# lib/watermarkr_web/live/video_live/index.ex

[...]
  @impl true
  def handle_info({:video_updated, _video_id}, socket) do
    {:noreply, assign(socket, :videos, list_videos())}
  end

  defp list_videos do
    Media.list_videos()
  end

end
```

There's, of course, a lot of room for optimization here since we don't need to fetch the whole list of videos every time there's a little update, but I'll leave that exercise to you, dear reader. :)

## Video Playback and Thumbnails

Now that we have all the info from Mux snug in our database, we can use that info to show some thumbnails in our index view and then even (drumroll) play the video.

Let's head to our index template and update the videos table to show a thumbnail if we have a video or a placeholder if nothing's there yet. We'll also remove the asset and playback ids from the table that have been added by our generators earlier:

```elixir
# lib/watermarkr_web/live/video_live/index.html.heex

[...]
<.table id="videos" rows={@videos} row_click={&JS.navigate(~p"/videos/#{&1}")}>
  <:col :let={video} label="Thumbnail">
    <img :if={video.playback_id} src={"https://image.mux.com/#{video.playback_id}/thumbnail.webp"} width="120" />
    <img :if={!video.playback_id} src="https://via.placeholder.com/120x68" width="120" />
  </:col>
  <:col :let={video} label="Title"><%= video.title %></:col>

  <:action :let={video}>
[...]
```

![Our updated video list](/img/videoupload-screen6-liveupdates.png)

For the final (and arguably most fun) piece of the puzzle, let us bring in the new [Mux Player](https://www.mux.com/player). The easiest way to do that is to load it from a CDN inside our root layout:

```html
# lib/watermarkr_web/components/layouts/root.html.heex
[...]
  <body class="bg-white antialiased">
    <%= @inner_content %>
  </body>
  <script src='https://cdn.jsdelivr.net/npm/@mux/mux-player'></script>
</html>
```

Now, all that's left is adding it to our show template:

```elixir
# lib/watermarkr_web/live/video_live/show.html.heex

[...]
  </:actions>
</.header>


<mux-player
  :if={@video.playback_id}
  stream-type="on-demand"
  playback-id={@video.playback_id}
  metadata-video-title={@video.title}
></mux-player>

[...]
```

And there you have it: a working upload for large video files, a mighty fine encoding engine and API powered by Mux, and modern video playback that works on all platforms. Boom!
In the next post, We'll explore how watermarking with Mux works, as I'm curious myself. **Always be learning!**

