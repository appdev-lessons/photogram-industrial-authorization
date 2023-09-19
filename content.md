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

## Finding holes

The project for today is _Photogram Industrial Authorization_.

Our industrial-grade application is shaping up, but it's sorely lacking in one area: security.

Right now, if you spin up the app, everyone can see everything and do everything (especially if they can guess URLs, which are easy to guess since we're following RESTful conventions and using sequential integer IDs). Try it out:

- `bin/dev`
- `rake sample_data`
- Sign in with `alice@example.com` / `password`.
- Poke around. Make a list of security issues that you discover.

We should make the application more secure! 

Here is our target:

[industrial-auth-1.matchthetarget.com](https://industrial-auth-1.matchthetarget.com/)

Did you spend some time looking for security holes? Try and do that and _think_ about the app before you keep reading.

---

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
- No limit to password attempts
- The `own_id` can be changed by manipulating the edit form for a photo and comment

Rails does a lot of helpful things for us like CSRF authenticity tokens, encrypting `session` so it can't be spoofed, etc., but it can't save us from ourselves! If we just use `scaffold` and leave all the routes, then they will be available.

## Tasks

Secure your application until you think it matches the target:

[industrial-auth-1.matchthetarget.com](https://industrial-auth-1.matchthetarget.com/) 

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

### Workflow

Try to use a good git workflow, including branching, opening Pull Requests, and merging back to `main` before starting on your next task. If you do, then it will be easy for us to give you feedback on each task in isolation.

### Resources

We have to think through each and every route and action that a user can take. Many of the security holes can be solved with:

 - [Filters](https://guides.rubyonrails.org/action_controller_overview.html#filters): `before_action` and `skip_before_action`.
 - [Redirecting](https://api.rubyonrails.org/v6.1.0/classes/ActionController/Redirecting.html): `redirect_to` and `redirect_back`.
 - Devise's `current_user` method.
 - Ruby's `if`/`else` statements.
 - Deleting or limiting routes with `only:` and `except:` after `resources`.


Now, spend some time trying to resolve the security holes on your own! When you get stuck or want some guidance, revisit the rest of the lesson below.

---

## `except` and `only` to Limit Routes, 00:55:00 to 01:02:00

Let's have a look at adding `except:` to the `:follow_requests` routes:

```ruby
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
{: mark_lines="9"}

This addition limits the routes, and removes five of them. But what happens now if we try to visit another user and click on "Follow" or "Un-Follow"? An error `undefined method follow_requests_path`!

We can trace this error in the code and see that it's because the `follow_requests/_form.html.erb` is triggering the `create` action, which we `except`ed in our list. Oops!

```ruby
# config/routes.rb

Rails.application.routes.draw do
  ...
  resources :comments
  resources :follow_requests, except: [:index, :show, :new, :edit]
  resources :likes
  resources :photos
  ...
```
{: mark_lines="6"}

We're left with `except`ing the four GET (`get`) routes. We only want the create, update, and destroy part of this resource, but not the read.

How about `likes`? Would we every want to show all likes, or edit likes? No! We `only` need a few routes for this resource:

```ruby
# config/routes.rb

Rails.application.routes.draw do
  ...
  resources :comments
  resources :follow_requests, except: [:index, :show, :new, :edit]
  resources :likes, only: [:create, :destroy]
  resources :photos
  ...
```
{: mark_lines="7"}

Using `except` and `only` in this manner is one of the first steps you should take to plug security holes.

For `photos`, we need most of the routes, except an index:

```ruby
# config/routes.rb

Rails.application.routes.draw do
  ...
  resources :comments
  resources :follow_requests, except: [:index, :show, :new, :edit]
  resources :likes, only: [:create, :destroy]
  resources :photos, except: [:index]
  ...
```
{: mark_lines="8"}

But, what happens now when you delete a photo in the app? It turns out you can delete _any_ photos in the app, not just your own.

## Authorization in Controller with `before_action` 01:02:00 to 01:12:00

One level below the routing is the controller. To fix the photo deletion, we'll need to drop down there to the `destroy` action:

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  ...
  # DELETE /photos/1 or /photos/1.json
  def destroy
    @photo.destroy

    respond_to do |format|
      format.html { redirect_back fallback_location: root_url, notice: "Photo was successfully destroyed." }
      format.json { head :no_content }
    end
  end
  ...
end
```
{: mark_lines="6"}

We can wrap the content of the action in an `if` statement dependent on the `current_user`'s ownership:

```ruby
  ...
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
  ...
```
{: mark_lines="3 10-12"}

Now the user will be sent back to the previous location (with a `notice` message) if they try to delete a photo that's not theirs.

The code, or some similar variation, that we would want to use over in other places is:

```ruby
if current_user == @photo.owner
```

Rails gives us a tool for that! Remember, we put the method `set_photo` in our controller in the `private` section and then used the `before_action` method to call this before the actions that required the current photo:

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  before_action :set_photo, only: %i[ show edit update destroy ]
  ...
  private
    # Use callbacks to share common setup or constraints between actions.
    def set_photo
      @photo = Photo.find(params[:id])
    end
  ...
end
```
{: mark_lines="4 8"}

We can add another similar action to check the ownership on some of our other actions:

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  before_action :set_photo, only: %i[ show edit update destroy ]
  before_action :ensure_current_user_is_owner, only: [:destroy, :update, :edit]
  ...
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
  ...
end
```
{: mark_lines="5 13-17"}

And we could revert the `destroy` action back to its original state (without the `if` statement).

To reiterate:

  - First we ask what routes we actually want and filter them from the `routes.rb` with `except` or `only`
  - For the remaining routes, we ask who is allowed to do what on each route.

## Conditionals in the View Templates 01:12:00 to 01:48:30

The next step in this process, is to hide links that aren't available to a given user. For instance, if a photo is now owned by someone, then they shouldn't even see the edit or delete links (which are Font Awesome icons in our case).

This is where conditional `if` statements in the view templates come in! Try and add those in yourself to hide things like the photo editing buttons, and lists of follow requests.

Let's do one such conditional together:

```erb
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
...
```
{: mark_lines="12 20"}

We are checking that the `current_user` is the `photo.owner` in the view template, and only rendering the Font Awesome links if they are.

## Hiding Private Users 01:48:30 to 01:55:00

Try to navigate to a user that is set to "Private". It turns out we can still see their whole page even if we don't follow them! Let's fix that.

We'll need a conditional on the user's show page:

```erb
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
{: mark_lines="9 23"}

But what should we use for the `truevalue` condition to selectively hide the user's navigation links and photos?

How about: `current_user.leaders.include?(@user)`? Do you follow the logic? We check if the `current_user` has an accepted follow request from the page of the user they are on (`@user`).

But we'll also need to allow viewing if the user is public, so we might need an "or" `||` statement:

```erb
<% if !@user.private? || current_user.leaders.include?(@user) %>
```

Now `@user.private?` is going to be true or false. The preceeding `!` asks "if not", flipping the true or false. In other words saying, "if the user is not private then render the page".

Try that conditional out? It seems to work on various public and private pages, but what happens when you visit your own page? Nothing there!

Well, we can just add another `||` or statement to our conditional:

```erb
<% if current_user == @user || !@user.private? || current_user.leaders.include?(@user) %>
```

Kind of a mouthful! But it works.

<aside markdown="1">
What if you had a user with administrative privilege? And they had a view template that needed to look completely different than a regular user? Would we want to embed dozens of conditionals to re-use the page for regular users and admins? In such cases, what we usually do instead is to render two entirely separate view templates to avoid excessive control flow in a single view.
</aside>

## Commenting on a Photo 01:55:00 to 02:08:30

Users should only be able to comment on photos in our app if they are following the user or the user is publich. Is that the case now? We basically need the same three-part conditional as in the user's show page.

Also, in the action, we want to prevent a user from creating or deleting if they aren't allowed to. We may have hid the route and/or content, but we should also bounce them from actions they aren't allowed in.

In the likes, we could use the `before_action:` similar to before (noting that `destroy` and `create` are the only routes we even created in our `resources`):

```ruby
# app/controllers/comments_controller.rb

class CommentsController < ApplicationController
  before_action :set_comment, only: %i[ show edit update destroy ]
  before_action :is_an_authorized_user, only: [:destroy, :create]
  ...
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
  ...
end
```
{: mark_lines="5 13-17"}

That copy-paste of the conditional won't work, because we don't have `@user` here, we have `@comment`. We need to get from `@comment` to the owner of it! Turns out we'll need to add another association accessor first!

```ruby
# app/models/comment.rb

...
class Comment < ApplicationRecord
  belongs_to :author, class_name: "User", counter_cache: true
  belongs_to :photo, counter_cache: true
  has_one :owner, through: :photo

  validates :body, presence: true
end
```
{: mark_lines="7"}

Now, back in the `comments_controller.rb`, we can put in the correct conditional:

```ruby
# app/controllers/comments_controller.rb

class CommentsController < ApplicationController
  before_action :set_comment, only: %i[ show edit update destroy ]
  before_action :is_an_authorized_user, only: [:destroy, :create]
  ...
  private
    # Use callbacks to share common setup or constraints between actions.
    def set_comment
      @comment = Comment.find(params[:id])
    end

    def is_an_authorized_user
      if current_user == @comment.owner || !@comment.owner.private? || current_user.leaders.include?(@comment.owner)
        redirect_back fallback_location: root_url, alert: "Not authorized"
      end
    end
  ...
end
```
{: mark_lines="14"}

We could test that, but we would unfortunately end up getting a `nil` returned for `@comment` when we try to `create` or `destroy`. Why? Because the `before_action` `:set_comment` is not being run before our new `is_authorized_user` method (the new method is not in the `only` list for `:set_comment`).

Actually, if we tried to debug this, we would quickly realize a flaw in our logic up to this point! We don't want to be looking at the owner of the `comment`! We want the owner of the `photo` that is being commented on. Oops. Let's change our code up:

```ruby
# app/controllers/comments_controller.rb

class CommentsController < ApplicationController
  before_action :set_comment, only: %i[ show edit update destroy ]
  before_action :is_an_authorized_user, only: [:destroy, :create]
  ...
  private
    # Use callbacks to share common setup or constraints between actions.
    def set_comment
      @comment = Comment.find(params[:id])
    end

    def is_an_authorized_user
      @photo = Photo.find(params.fetch(:comment).fetch(:photo_id))
      if current_user != @photo.owner && @photo.owner.private? && !current_user.leaders.include?(@photo.owner)
        redirect_back fallback_location: root_url, alert: "Not authorized"
      end
    end
  ...
end
```
{: mark_lines="13 14"}

First we need to get the current photo from the `params` hash (from `comment[photo_id]`), then we need to change the logic in our conditional (switching the location of the `!` "if not" modifiers, and changing `||` to `&&`) to get what we want: Users can only comment of public photos, or photos of their leaders.

## Tests 02:08:30 to 02:10:00

Look at all of the steps we need to go through to test fixing these security holes. It is tedious to click through the app and test everything. This is the moment you should be think, "I should write some tests".

Testing pays off big when it comes to security and authorization, but it is time consuming. We won't do it together now, but just keep this in mind.

## Industrial Authentication 2 with `pundit` 02:12:00 to 02:35:00

**BENP: this chapter: https://chapters.firstdraft.com/chapters/860 is woven in below**

Now that've got a handle on, fundamentally, how authorization is done — with conditionals and redirects — for an application of any real size, it's probably worth using an authorization library. A good one will give us helper methods that will allow us to be more concise, and more importantly will help us avoid letting security holes slip through the cracks.

There are many Ruby authorization libraries, but my go-to is one called [Pundit](https://github.com/varvet/pundit). There's a workspace prepared to help us experiment with this library, called [Industrial Authentication 2](https://github.com/appdev-projects/industrial-auth-2). Open that in Gitpod:

  - add `gem "pundit"` to your `Gemfile` 
  - `bundle` 
  - Get the app running with `sample_data` and `bin/server`.

Pundit revolves around the idea of **policies**. Create a folder within `app/` called `policies/`, and within it create a file called `photo_policy.rb`. This will be a plain ol' Ruby object (PORO):

```ruby
# app/policies/photo_policy.rb

class PhotoPolicy
end
```

The idea is that we want to encapsulate all knowledge about who can do what with a photo inside instance methods within this "policy" class. Let's set up the class to accept a user and a photo when instantiated:

```ruby
# app/policies/photo_policy.rb

class PhotoPolicy
  attr_reader :user, :photo

  def initialize(user, photo)
    @user = user
    @photo = photo
  end
end
```
{: mark_lines="4 6-9"}

And now, for example, to figure out who can see a photo, let's define a method called `show?` that will return `true` if the user is allowed to see the photo and `false` if not:

<aside markdown="1">
A question mark at the end of a method name doesn't do anything special, functionally; it's just another letter in the method name, as far as Ruby is concerned.

It's a convention among Rubyists to end the names of methods that return `true` or `false` with a question mark, so I'm following that convention here with `show?`.
</aside>

```ruby
# app/policies/photo_policy.rb

class PhotoPolicy
  attr_reader :user, :photo

  def initialize(user, photo)
    @user = user
    @photo = photo
  end

  # Our policy is that a photo should only be seen by the owner or followers
  #   of the owner, unless the owner is not private in which case anyone can
  #   see it
  def show?
    user == photo.owner ||
      !photo.owner.private? ||
      photo.owner.followers.include?(user)
  end
end
```
{: mark_lines="11-18"}

Let's test it out in `rails console`.

First we'll get two users:

```ruby
[1] pry(main)> alice = User.first
=> #<User id: 15>
[2] pry(main)> bob = User.second
=> #<User id: 16>
```

Then we'll see if Bob follows Alice (Bob shouldn't, so unfollow Alice if `followers.include?` returns true), and if Alice is set to private (again, if this is not the case, then set Alice to private): 

```ruby
[3] pry(main)> alice.followers.include?(bob)
=> false
[4] pry(main)> alice.private?
=> true
```

Now we get get one of Alice's photos:

```ruby
[5] pry(main)> photo = alice.own_photos.first
=> #<Photo id: 157>
```

First, we instantiate a policy for Alice, `policy_a`, and based on our `.show?` method, we check the visibility:

```ruby
[6] pry(main)> policy_a = PhotoPolicy.new(alice, photo)
=> #<PhotoPolicy:0x00007fca9886d8e0>
[7] pry(main)> policy_a.show?
=> true
```

And we do the same for Bob:

```ruby
[8] pry(main)> policy_b = PhotoPolicy.new(bob, photo)
=> #<PhotoPolicy:0x00007fca5fa3a6e8>
[9] pry(main)> policy_b.show?
=> false
```

Great! Now we can replace the conditionals that we have scattered about the application with this method. 

Let's start by locking down access to the `Photos#show` action. 

### Using `before_action`

We can start by following a similar technique as before, and utilizing our new policy in a `before_action`:

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  before_action :set_photo, only: %i[ show edit update destroy ]
  before_action :ensure_current_user_is_owner, only: [:destroy, :update, :edit]
  before_action :ensure_user_is_authorized, only: [:show]
  ...
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

    def ensure_user_is_authorized
      if !PhotoPolicy.new(current_user, @photo).show?
        redirect_back fallback_location: root_url
      end
    end
  ...
end
```
{: mark_lines="6 20-24"}

(Mind the `!` before `...show?`; we want to redirect if the user _can't_ view the photo.)

Now we can try to navigate to a private user's photo, and we should be bounced.

### Raising An Exception

Instead of redirecting, let's take a slightly different tack: we'll raise an exception if the `current_user` isn't authorized, so that we can get an error message. 

We want to get error messages, because soon you will be setting up error monitoring systems. These errors would get logged and tracked automatically for us, so we could keep track of them for debugging.

Pundit provides a specific error message class for exactly this purpose: `Pundit::NotAuthorizedError`.

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  ...
  before_action :ensure_user_is_authorized, only: [:show]
  ...
    def ensure_user_is_authorized
      if !PhotoPolicy.new(current_user, @photo).show?
        raise Pundit::NotAuthorizedError, "not allowed"
      end
    end
  ...
end
```
{: mark_lines="9"}

Test it out by visiting a show page that you shouldn't be able to (you can remove the `/edit` from an edit page to find a URL.)

Great! But what if we don't want to show an error page? Redirecting with a flash message was pretty nice. Well, we can rescue that specific exception in `ApplicationController`:

```ruby
# app/controllers/application_controller.rb

class ApplicationController
  ...
  rescue_from Pundit::NotAuthorizedError, with: :user_not_authorized

  private

    def user_not_authorized
      flash[:alert] = "You are not authorized to perform this action."
      
      redirect_back fallback_location: root_url
    end
end
```
{: mark_lines="5 7 9-13"}

Now we have all of the same functionality back, but we're nicely set up to encapsulate all of our authorization logic in one place.

## Convention over configuration in controllers 02:35:00 to 02:38:30

So far, Pundit hasn't done anything for us at all, other than providing the exception class that we raised. We wrote all the Ruby ourselves. But now, let's use some some helper methods from Pundit that will allow us to be _very_ concise, if we follow conventional naming.

First, `include Pundit` in `ApplicationController` to gain access to the methods in all of the other controllers:

```ruby
# app/controllers/application_controller.rb

class ApplicationController
  include Pundit
...
```
{: mark_lines="4"}

Now, we can use the `authorize` method. Instead of all this:

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  ...
  before_action :ensure_user_is_authorized, only: [:show]
  ...
    def ensure_user_is_authorized
      if !PhotoPolicy.new(current_user, @photo).show?
        raise Pundit::NotAuthorizedError, "not allowed"
      end
    end
  ...
end
```
{: mark_lines=""}

We can write directly in the `show` action, without any `before_action` step, just this:

```ruby
# app/controllers/photos_controller.rb

class PhotosController < ApplicationController
  ...
  def show
    authorize @photo
  end
  ...
end
```
{: mark_lines="6"}

🤯 What just happened? When you pass the `authorize` method an instance of `Photo`:

 - It assumes there is a class called `PhotoPolicy` in `app/policies`.
 - It assumes there is a method called `current_user`.
 - It passes `current_user` as the first argument and whatever you pass to `authorize` as the second argument to a new instance of `PhotoPolicy`.
 - It calls a method named after the action with a `?` appended on the new policy instance.
 - If it gets back `false`, it raises `Pundit::NotAuthorizedError`.

## Views with Pundit 02:38:30 to 02:42:00

In view templates, we now have a `policy` helper method that will make it easier to conditionally hide and show things. For example, assuming we define a `show?` method in our user's policy, like so:

```ruby
# app/policies/user_policy.rb

class UserPolicy
  attr_reader :current_user, :user

  def initialize(current_user, user)
    @current_user = current_user
    @user = user
  end

  def show?
    user == current_user ||
     !user.private? || 
     user.followers.include?(current_user)
  end
end
```
{: mark_lines=""}

Then we can head to the view template and change this conditional:

```erb
<!-- app/views/users/show.html.erb -->

...
<% if current_user == @user || !@user.private? || current_user.leaders.include?(@user) %>
  ...
<% end %>
```
{: mark_lines="4"}

to:

```erb
<!-- app/views/users/show.html.erb -->

...
<% if policy(@user).show? %>
  ...
<% end %>
```
{: mark_lines="4"}

## Just plain ol' Ruby

Since policies are just POROs, we can bring all our Ruby skills to bear: inheritance, [aliasing](https://medium.com/rubycademy/alias-in-ruby-bf89be245f69){:target="_blank"}, etc.

To start with, we can run the generator `rails g pundit:install` which creates a good starting point policy to inherit from in `app/policies/application_policy.rb`. Take a look and see what you think. If you like it, let's inherit from it:

```ruby
# app/policies/photo_policy.rb

class PhotoPolicy < ApplicationPolicy
```

## Secure by default 02:42:00 02:47:00

So — that means that all we need to do from now on is:

 1. Remember to define a method in our policy that matches **every** action name (but we can alias).
 2. Remember to call `authorize` within **every** action.

If we've done #2, which will force us to do #1, that will ensure that we've at least thought about who can get into every action.

Try visiting `/follow_requests` right now — oops! Quite a big security hole, and one that's depressingly common to find left open after a resource has been generated with `scaffold`.

Pundit includes a method called `verify_authorized` that we can call in an `after_action` in the `ApplicationController` to help enforce the discipline of pruning our unused routes:

```ruby
# app/controllers/application_controller.rb

class ApplicationController
  include Pundit
  
  after_action :verify_authorized
  ...
```
{: mark_lines="6"}

[There's another method called `policy_scope`](https://github.com/varvet/pundit#scopes){:target="_blank"} similar to `authorize` that's used for collections rather than single objects. Usually, you'll want to ensure that either one or the other is called with something like the following in `ApplicationController`:

```ruby
# app/controllers/application_controller.rb
after_action :verify_authorized, except: :index
after_action :verify_policy_scoped, only: :index
```

Now try visiting `/follow_requests` or some other scaffolded route that was insecurely left reachable. You'll see that you can't.

If necessary, you can make the choice to `skip_before_action :verify_authorized` on a case-by-case basis, as we did for `:authenticate_user!`. We are now secure-by-default instead of insecure-by-default.

## Read more

There's more to Pundit [that you should read about in the README](https://github.com/varvet/pundit){:target="_blank"}, but not a _ton_ more. That's what I like about it — it's relatively lightweight, but gets the job done well. The way that it is structured also plays great with [Rails I18n](https://guides.rubyonrails.org/i18n.html){:target="_blank"} and other libraries. Another powerful tool for your belt.
