# Photogram Industrial Authorization

## Walkthrough video

<div class="bg-red-100 py-1 px-5" markdown="1">
**Please note**, the video is from a previous iteration of the project, so there are some differences:

- I am using Gitpod as my cloud editor, so the interface looks a bit different.
- Anything contained in the project "README" or "chapters" is now contained in this Lesson
- I use `bin/server` to start my live app preview, _you_ should use `bin/dev`
- I use `rails sample_data`, _you_ should use `rake sample_data`
</div>

Did you read the differences above? Good! Then [here is a walkthrough video for this project.](https://share.descript.com/view/qqL5sX534E1)

**As you watch the video, pause it frequently, read the associated text, and type out the code.**

## Getting started

There are no automated tests for this project. Instead you will create a fork of two repositories as you work through this lesson, and open pull requests to be submitted on Canvas.

Here is the first repository to fork and open a codespace on:

[github.com/appdev-projects/industrial-auth-1/fork](https://github.com/appdev-projects/industrial-auth-1/fork)

A potential target to compare your app to can be found here:

[industrial-auth-1.matchthetarget.com](https://industrial-auth-1.matchthetarget.com/)

But we won't rely much on the target, rather, we want to think about how we can make _our app_ more secure.

## Finding holes

The project for today is _Photogram Industrial Authorization_.

Our industrial-grade application is shaping up, but it's sorely lacking in one area: security.

Right now, if you spin up the app, everyone can see everything and do everything (especially if they can guess URLs, which are easy to guess since we're following RESTful conventions and using sequential integer IDs). Try it out:

- `bin/dev`
- `rake sample_data`
- Sign in with `alice@example.com` / `password`.
- Poke around. Make a list of security issues that you discover.

We should make the application more secure! 

Here's just _some_ of the security issues we need to fix:

- A user can edit any other user's photos, captions, and comments, including deleting them
- A user can see, accept, and reject other user's follow requests
- A user can guess at URL endpoints and find them, even if they aren't linked, such as: 
  - `/comments`
  - `/photos`
  - `/likes`
  - `/follow_requests`
  - That implies they can also reach `/ID/edit`, etc. for each of those
- Private profiles are not private at all

Rails does a lot of helpful things for us like CSRF authenticity tokens, encrypting `session` so it can't be spoofed, etc., but it can't save us from ourselves! If we just use `scaffold` and leave all the routes, then they will be available.

## Tasks

A non-exhaustive list of things to do:

 - If I click delete on a photo that's not mine, I should be redirected back to the page I was on before. *(Once you've locked that down properly, you should hide the link so as not to be rude.)*
 - If I click edit on a photo that's not mine, I should be redirected back to the page I was on before. *(Once you've locked that down properly, you should hide the link so as not to be rude.)*
 - If I click delete on a comment that's not mine, I should be redirected back to the page I was on before. *(Once you've locked that down properly, you should hide the link so as not to be rude.)*
 - If I click edit on a comment that's not mine, I should be redirected back to the page I was on before. *(Once you've locked that down properly, you should hide the link so as not to be rude.)*
 - Regardless of following/private status, I can visit any user's profile page, see their followers, and who they are following. But I shouldn't be able to see the posts and likes section at the bottom of their profile page unless:
    - They are not private. Then anyone can see their posts and likes, even if they are not a follower.
    - They are private and I am an accepted follower. (I can still see anyone's post if it e.g. shows up in my Discover, and that is how I might wind up on their profile page to send them a follow request.)
 - Only a user should be able to see their own Pending follow requests.
 - The collections of photos that a user sees through their leaders (feed, discover) should only be viewable by them. In other words, `/carol/feed` and `/carol/discover` should not be visitable by Alice or Bob, only by Carol.

## Resources

We have to think through each and every route and action that a user can take. Many of the security holes can be solved with:

 - [Filters](https://guides.rubyonrails.org/action_controller_overview.html#filters): `before_action` and `skip_before_action`.
 - [Redirecting](https://api.rubyonrails.org/v6.1.0/classes/ActionController/Redirecting.html): `redirect_to` and `redirect_back`.
 - Devise's `current_user` method.
 - Ruby's `if`/`else` statements.
 - Deleting or limiting routes with `only:` and `except:` after `resources`.

## `except` and `only` to limit routes

Let's have a look at adding `except:` to the `:follow_requests` routes:

```ruby{9}
# config/routes.rb

Rails.application.routes.draw do
  root "users#feed"

  devise_for :users
  
  resources :comments
  resources :follow_requests, except: [:index, :show, :new, :edit, :create]
  resources :likes
  resources :photos

  get ":username" => "users#show", as: :user
  get ":username/liked" => "users#liked", as: :liked
  get ":username/feed" => "users#feed", as: :feed
  get ":username/discover" => "users#discover", as: :discover
end
```

This addition limits the routes, and removes five of them. But what happens now if we try to visit another user and click on "Follow" or "Un-Follow"? An error `undefined method follow_requests_path`!

We can trace this error in the code and see that it's because the `follow_requests/_form.html.erb` is triggering the `create` action, which we `except`ed in our list. Oops!

```ruby{6}
# config/routes.rb

Rails.application.routes.draw do
  # ...
  resources :comments
  resources :follow_requests, except: [:index, :show, :new, :edit]
  # ...
```

We're left with `except`ing the four GET (`get`) routes. We only want the create, update, and destroy part of this resource, but not the read.

How about `likes`? Would we every want to show all likes, or edit likes? No! We `only` need a few routes for this resource:

```ruby{6}
# config/routes.rb

Rails.application.routes.draw do
  # ...
  resources :follow_requests, except: [:index, :show, :new, :edit]
  resources :likes, only: [:create, :destroy]
  # ...
```

Using `except` and `only` in this manner is one of the first steps you should take to plug security holes.

For `photos`, we need most of the routes, except an index:

```ruby{7}
# config/routes.rb

Rails.application.routes.draw do
  # ...
  resources :follow_requests, except: [:index, :show, :new, :edit]
  resources :likes, only: [:create, :destroy]
  resources :photos, except: [:index]
  # ...
```

But, what happens now when you delete a photo in the app? It turns out you can delete _any_ photos in the app, not just your own.

## Authorization in controller with `before_action`

One level below the routing is the controller. To fix the photo deletion, we'll need to drop down there to the `destroy` action:

```ruby{5}
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  # ...
  def destroy
    @photo.destroy

    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Photo was successfully destroyed." }
      format.json { head :no_content }
    end
  end
  # ...
end
```

We can wrap the content of the action in an `if` statement dependent on the `current_user`'s ownership:

```ruby{3,10-12}
  # ...
  def destroy
    if current_user == @photo.owner
      @photo.destroy

      respond_to do |format|
        format.html { redirect_back fallback_location: root_url, notice: "Photo was successfully destroyed." }
        format.json { head :no_content }
      end
    else
      redirect_back(fallback_location: root_url, notice: "Nice try, but that is not your photo.")
    end
  end
  # ...
```

Now the user will be sent back to the previous location (with a `notice` message) if they try to delete a photo that's not theirs.

The code, or some similar variation, that we would want to use over in other places is:

```ruby
if current_user == @photo.owner
```

Rails gives us a tool for that! Remember, we put the method `set_photo` in our controller in the `private` section and then used the `before_action` method to call this before the actions that required the current photo:

```ruby{4,8}
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  before_action :set_photo, only: %i[ show edit update destroy ]
  # ...
  private
    # Use callbacks to share common setup or constraints between actions.
    def set_photo
      @photo = Photo.find(params[:id])
    end
  # ...
end
```

We can add another similar action to check the ownership on some of our other actions:

```ruby{5,13-17}
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  before_action :set_photo, only: %i[ show edit update destroy ]
  before_action :ensure_current_user_is_owner, only: [:destroy, :update, :edit]
  # ...
  private
    # Use callbacks to share common setup or constraints between actions.
    def set_photo
      @photo = Photo.find(params[:id])
    end

    def ensure_current_user_is_owner
      if current_user != @photo.owner
        redirect_back fallback_location: root_url, alert: "You're not authorized for that."
      end
    end
  # ...
end
```

And we could revert the `destroy` action back to its original state (without the `if` statement).

To reiterate:

  - First we ask what routes we actually want and filter them from the `routes.rb` with `except` or `only`
  - For the remaining routes, we ask who is allowed to do what on each route.

## Conditionals in the view templates

The next step in this process, is to hide links that aren't available to a given user. For instance, if a photo is not owned by someone, then they shouldn't even see the edit or delete links (which are Font Awesome icons in our case).

This is where conditional `if` statements in the view templates come in! Try and add those in yourself to hide things like the photo editing buttons, and lists of follow requests.

Let's do one such conditional together:

```erb{12,20}
<!-- app/views/photos/_photo.html.erb -->

<div class="card">
  <div class="card-body py-3 d-flex align-items-center justify-content-between">
    <h2 class="h5 m-0 p-0 d-flex align-items-center">
      <%= image_tag photo.owner.avatar_image, class: "rounded-circle mr-2", width: 36 %>

      <%= link_to photo.owner.username, user_path(photo.owner.username), class: "text-dark" %>
    </h2>

    <div>
      <% if current_user == photo.owner %>
        <%= link_to edit_photo_path(photo), class: "btn btn-link btn-sm text-muted" do %>
          <i class="fas fa-edit fa-fw"></i>
        <% end %>

        <%= link_to photo, method: :delete, class: "btn btn-link btn-sm text-muted" do %>
          <i class="fas fa-trash fa-fw"></i>
        <% end %>
      <% end %>
      
    </div>
  </div>
<!-- ... -->
```

We are checking that the `current_user` is the `photo.owner` in the view template, and only rendering the Font Awesome links if they are.

## Hiding private users

Try to navigate to a user that is set to "Private". It turns out we can still see their whole page even if we don't follow them! Let's fix that.

We'll need a conditional on the user's show page:

```erb{9,23}
<!-- app/views/users/show.html.erb -->

<div class="row mb-4">
  <div class="col-md-6 offset-md-3">
    <%= render "users/user", user: @user %>
  </div>
</div>

<% if truevalue %>
  <div class="row mb-2">
    <div class="col-md-6 offset-md-3">
      <%= render "users/profile_nav", user: @user %>
    </div>
  </div>

  <% @user.own_photos.each do |photo| %>
    <div class="row mb-4">
      <div class="col-md-6 offset-md-3">
        <%= render "photos/photo", photo: photo %>
      </div>
    </div>
  <% end %>
<% end %>
```

But what should we use for the `truevalue` condition to selectively hide the user's navigation links and photos?

How about: `current_user.leaders.include?(@user)`? Do you follow the logic? We check if the `current_user` has an accepted follow request from the page of the user they are on (`@user`).

But we'll also need to allow viewing if the user is public, so we might need an "or" `||` statement:

```erb
<% if !@user.private? || current_user.leaders.include?(@user) %>
```

Now `@user.private?` is going to be true or false. The preceding `!` asks "if not", flipping the true or false. In other words saying, "if the user is not private then render the page".

Try that conditional out? It seems to work on various public and private pages, but what happens when you visit your own page? Nothing there!

Well, we can just add another `||` or statement to our conditional:

```erb
<% if current_user == @user || !@user.private? || current_user.leaders.include?(@user) %>
```

Kind of a mouthful! But it works.

<aside markdown="1">
What if you had a user with administrative privilege? And they had a view template that needed to look completely different than a regular user? Would we want to embed dozens of conditionals to re-use the page for regular users and admins? In such cases, what we usually do instead is to render two entirely separate view templates to avoid excessive control flow in a single view.
</aside>

## Commenting on a photo

Users should only be able to comment on photos in our app if they are following the user or the user is public. Is that the case now? We basically need the same three-part conditional as in the user's show page.

Also, in the action, we want to prevent a user from creating or deleting if they aren't allowed to. We may have hid the route and/or content, but we should also bounce them from actions they aren't allowed in.

In the likes, we could use the `before_action:` similar to before (noting that `destroy` and `create` are the only routes we even created in our `resources`):

```ruby{5,13-17}
# app/controllers/comments_controller.rb

class CommentsController < ApplicationController
  before_action :set_comment, only: %i[ show edit update destroy ]
  before_action :is_an_authorized_user, only: [:destroy, :create]
  # ...
  private
    # Use callbacks to share common setup or constraints between actions.
    def set_comment
      @comment = Comment.find(params[:id])
    end

    def is_an_authorized_user
      if current_user == @user || !@user.private? || current_user.leaders.include?(@user)
        redirect_back fallback_location: root_url, alert: "Not authorized"
      end
    end
  # ...
end
```

That copy-paste of the conditional won't work, because we don't have `@user` here, we have `@comment`. We need to get from `@comment` to the owner of it! Turns out we'll need to add another association accessor first!

```ruby{6}
# app/models/comment.rb

class Comment < ApplicationRecord
  belongs_to :author, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true
  has_one :owner, through: :photo

  validates :body, presence: true
end
```

Now, back in the `comments_controller.rb`, we can put in the correct conditional:

```ruby{8}
# app/controllers/comments_controller.rb

class CommentsController < ApplicationController
  before_action :set_comment, only: %i[ show edit update destroy ]
  before_action :is_an_authorized_user, only: [:destroy, :create]
  # ...
    def is_an_authorized_user
      if current_user == @comment.owner || !@comment.owner.private? || current_user.leaders.include?(@comment.owner)
        redirect_back fallback_location: root_url, alert: "Not authorized"
      end
    end
  # ...
end
```

We could test that, but we would unfortunately end up getting a `nil` returned for `@comment` when we try to `create` or `destroy`. Why? Because the `before_action` `:set_comment` is not being run before our new `is_authorized_user` method (the new method is not in the `only` list for `:set_comment`).

Actually, if we tried to debug this, we would quickly realize a flaw in our logic up to this point! We don't want to be looking at the owner of the `comment`! We want the owner of the `photo` that is being commented on. Oops. Let's change our code up:

```ruby{8,9}
# app/controllers/comments_controller.rb

class CommentsController < ApplicationController
  before_action :set_comment, only: %i[ show edit update destroy ]
  before_action :is_an_authorized_user, only: [:destroy, :create]
  # ...
    def is_an_authorized_user
      @photo = Photo.find(params.fetch(:comment).fetch(:photo_id))
      if current_user != @photo.owner && @photo.owner.private? && !current_user.leaders.include?(@photo.owner)
        redirect_back fallback_location: root_url, alert: "Not authorized"
      end
    end
  # ...
end
```

First we need to get the current photo from the `params` hash (from `comment[photo_id]`), then we need to change the logic in our conditional (switching the location of the `!` "if not" modifiers, and changing `||` to `&&`) to get what we want: Users can only comment of public photos, or photos of their leaders.

<div class="bg-blue-100 py-1 px-5" markdown="1">

**Tests**

Look at all of the steps we need to go through to test fixing these security holes. It is tedious to click through the app and test everything. This is the moment you should be think, "I should write some tests".

Testing pays off big when it comes to security and authorization, but it is time consuming. We won't do it together now, but just keep this in mind.
</div>

## Industrial authorization with `pundit`

Have you made some (painful) changes to your codebase to increase security? Once you've made a few patches, submit a pull request on Canvas, then visit the next lesson where you will learn some shortcuts to authorization with the `pundit` gem.
