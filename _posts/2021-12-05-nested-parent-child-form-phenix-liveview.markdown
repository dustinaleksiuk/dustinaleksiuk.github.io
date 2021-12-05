---
layout: post
title:  "Creating a Nested Parent-Child Form in Phoenix LiveView"
date:   2021-12-05 10:41:04 -0700
categories: liveview
---
## The Backstory

One of my first tasks recently as a new Elixir and Phoenix LiveView developer was to write a SPA-style form that would allow the user to add and remove child records from a parent and save the changes on a single page with a single save button. The user should also be able to cancel and abandon their changes. 

In other words, I want to add or remove a child row without going to the database until the user clicks the save button. When the user clicks save, all the form changes get saved at once, to both the parent and child table.

It is also important that validation errors show up on the correct row using the built-in functionality provided by the framework. No hackery. I wanted to work with the framework, not around it.

As a new LiveView user, I found this surprisingly difficult to learn to do. From researching, asking questions, and reading the [Elixir forum](https://elixirforum.com/), I learned I wasn't alone.

## The Sample App

The sample app is a LiveView website that stores albums and tracks for the album. The edit form lets the user edit the album, and add and remove tracks.

## Key Concepts and Functions

I'll go into more detail below, but these are the key points that opened the door to finally figuring out how to get it done.

#### [LiveView forms](https://hexdocs.pm/phoenix_live_view/form-bindings.html)

In LiveView, you want ONE submit button and ONE changeset for each form. That's just how it works. You can't have an individual submit button per child row because [the event can't tell you what button was pressed](https://github.com/phoenixframework/phoenix_live_view/issues/511#issuecomment-563243927). This is noteworthy because some of us are tempted to put a submit button on each child row, especially since the submit event passes the entire form params back to the event handler.

#### [inputs_for/2](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#inputs_for/2)

This is the function that lets you nest association data in a form. It is the mechanism that loops (recurses, officially) over the list of tracks and displays the form fields for each track.

#### [get_field/3](https://hexdocs.pm/ecto/Ecto.Changeset.html#get_field/3)

This little guy is a game changer. I was killing myself to try to get data from the changeset's changes and data fields based on a tutorial I was trying to follow. I swear this function is the best kept secret in Ecto because I asked various people (and read the only tutorial I could find on this problem).

## The Code

[This public github repository](https://github.com/dustinaleksiuk/recordstore) contains my little Record Store app that demonstrates this solution.

I chose to put the fields for the child tracks into their own LiveComponent. This isn't necessary for the solution but I think it makes the code organization a little nicer.

### Loading the Data

This is an edit form, so when the page loads we need to fetch the album that the user clicked on as well as the tracks. The important part here is to preload your tracks so that they are available as children to the album schema. I do the preload in the `RecordStore.Albums` context in `albums.ex`.

{% highlight elixir %}
def get_album!(id, true) do
  get_album!(id, false)
  |> Repo.preload(:tracks)
end
{% endhighlight %}

### The Album (Parent) Schema

The parent schema, album, is at `lib/recordstore/albums/album.ex`. I associate the child tracks to the album the usual way:

{% highlight elixir %}
has_many(:tracks, Track,
  foreign_key: :album_id,
  on_replace: :delete
)
{% endhighlight %}

We also need a cast_assoc for the tracks because we will be saving the tracks via the album schema:

{% highlight elixir %}
def changeset(album, attrs) do
  album
  |> cast(attrs, [:name, :artist, :genre, :rating])
  |> cast_assoc(:tracks, with: &Track.changeset/2)
  |> validate_required([:name, :rating, :genre])
end
{% endhighlight %}

### The Track (Child) Schema

The tracks are represented by the track schema in `lib/recordstore/albums/track.ex`. The only interesting thing here is a virtual temp_id field. This field will store a temporary ID to allow us to remove tracks that haven't been saved yet.

{% highlight elixir %}
field :temp_id, :string, virtual: true
{% endhighlight %}
    
### The LiveView

The LiveView file at `lib/recordstore_web/live/album_live/edit.ex` and its associated heex template are where all the action is.

#### Loading the Album

When the form loads and the album is loaded into memory as a struct, I loop through the tracks and set a temp id on each one so that we can remove unsaved tracks.

{% highlight elixir %}
defp preload_temp_ids(album) do
  updated_tracks = Enum.map(album.tracks, fn t -> %Track{t | temp_id: Ecto.UUID.generate()} end)
  %Album{album | tracks: updated_tracks}
end
{% endhighlight %}

#### Adding a Track

In the event for adding a track, we generate a new temp id, fetch the tracks list from the changeset, and add a new track with the new temp id to the list.

{% highlight elixir %}
def handle_event("add-track", _params, socket) do
  changeset = socket.assigns.changeset
  temp_id = Ecto.UUID.generate()

  tracks =
    Changeset.get_field(changeset, :tracks)
    |> Enum.concat([%Track{temp_id: temp_id}])

  changeset = Changeset.put_assoc(changeset, :tracks, tracks)
  {:noreply, assign(socket, :changeset, changeset)}
end
{% endhighlight %}

#### Removing a Track

To remove a track, the event needs to grab the list of tracks and remove the one that the user clicked on, and then set the list back onto the album changeset.

{% highlight elixir %}
def handle_event("remove-track", %{"temp-id" => temp_id}, %{assigns: assigns} = socket) do
  tracks =
    Changeset.get_field(assigns.changeset, :tracks)
    |> Enum.reject(&(&1.temp_id == temp_id))

  changeset = Changeset.put_assoc(assigns.changeset, :tracks, tracks)

  {:noreply, assign(socket, :changeset, changeset)}
end
{% endhighlight %}

#### The Change Event

The LiveView `phx-change` event fires on every action or keystroke in the app. Often all one would do here is run the form params through the changeset function so that the validation is up to date. However, when the last track is removed, the album form params no longer have the `tracks:` field, so the changeset doesn't know that there are changes.

To solve this, I added a `process_params` function that sets the `tracks:` field to an empty map. It is called at the top of the `change` event as well as the `save` event. Thanks to Rowland Carlson for finding this problem and submitting a fix.

*Note:* I'm mildly uncomfortable with this part of the code because I don't like having to mess with the form params unless absolutely necessary. However in this situation I think it's the most elegant solution.

{% highlight elixir %}
def handle_event("change", %{"album" => album_params}, socket) do
  album_params = album_params |> process_params()

  changeset =
    socket.assigns.album
    |> Albums.change_album(album_params)
    |> Map.put(:action, :validate)

  {:noreply, assign(socket, :changeset, changeset)}
end
{% endhighlight %}

This is the code that sets the empty map:

{% highlight elixir %}
defp process_params(album_params) do
  case album_params do
    %{"tracks" => _} -> album_params
    _ -> Map.put(album_params, "tracks", %{})
  end
end
{% endhighlight %}

#### The Heex Templates

The most important part in this area is to use [inputs_for/2](https://hexdocs.pm/phoenix_html/Phoenix.HTML.Form.html#inputs_for/2) to display the association data. I pulled this part out into a component stored in `tracks_component.html.heex` because it felt cleaner but it's not necessary.

Note the comprehension that loop over the tracks and displays each one. The `phx-click` event and `phx-value-temp-id` attributes for removing a track are seen here, as well as the `phx-click` attribute for adding a track. These bubble up to the LiveView in `edit.ex`.

{% highlight jsp %}
<div>
    Tracks component

    <div class="track-list">
        <%= for inner_form <- inputs_for(@form, :tracks) do %>
            <%= hidden_input inner_form, :temp_id %>
            <div>            
                <%= text_input inner_form, :name %>
                <%= error_tag inner_form, :name %>
            </div>
            <div>            
                <%= text_input inner_form, :position %>
                <%= error_tag inner_form, :position %>
            </div>
            <div>
                <a href="#" phx-click="remove-track" phx-value-temp-id={input_value(inner_form, :temp_id)}>&times;</a>
            </div>

        <% end %>
    </div>
    <a href="#" phx-click="add-track">Add a Track</a>
</div>
{% endhighlight %}

## Conclusion

Adding this functionality to our app was my first real task as a brand new LiveView developer. I found it difficult because one has to learn many aspects of LiveView to make it work. I hope that this little writeup saves someone else the headaches I experienced while figuring it all out.

If I've made any embarrassing mistakes or if there is an obviously better way to solve this requirement, I would love to know about it. Please post an issue at the [GitHub repo for the example code](https://github.com/dustinaleksiuk/recordstore). I'd also like to know of anything that was unclear about this writeup so I can improve it.

