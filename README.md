### Rails Full-Text Search Form with AJAX

My most popular post, with over 19,000 pageviews over 18 months, is [Create a Simple Search Form with Rails](http://www.rymcmahon.com/articles/2). It benefitted from a great Google search results page rank (usually top 3 for "rails search form") and from the fact that this is a common feature to add to a Rails app.

Many readers commented on the article and some even had suggestions for taking the feature further. To respond to many of those requests, I thought I'd write an updated post that demonstrates how to build a more sophisticated search and submit the form via AJAX.

Let's start by creating a simple blog app and we will add a form that allows visitors to search for articles by their title or body. I am using Rails 5.2, Ruby 2.4.1, and PostgreSQL 9.6.3.

#### The Setup

Create a new Rails app with PostgreSQL as the database:
```ruby
$ rails new blog --database=postgresql
```

Generate some scaffolding for the Post model with title and body attributes:

```ruby
$ rails g scaffold Post title:string body:text
```

Create and migrate the database:

```ruby
$ rails db:create && rails db:migrate
```

Start the Rails server and go to ```http://localhost:3000/posts/``` to render the Posts index page.

We are going to need a few gems—Faker for adding fake data to the development database, jQuery (it was removed from Rails in 5.0), and pg_search for using PostgreSQL’s full-text search. In the Gemfile add:
 ```ruby
gem 'faker', :git => 'https://github.com/stympy/faker.git', :branch => 'master'
gem 'jquery-rails'
gem 'pg_search'
```
Run ``` $ bundle install ```.

Open app/assets/javascripts/application.js and add ```//= require jquery3``` so jQuery is available in the asset pipeline. The file should look like this:

```ruby
//= require rails-ujs
//= require jquery3
//= require activestorage
//= require turbolinks
//= require_tree .
```
### Seed the Database

Let's add some records to the database using the Faker gem. Go to ```app/db``` , open ```seeds.rb``, and add:

```ruby
100.times do
  Post.create(
    title: Faker::Hipster.sentence,
    body: Faker::Hipster.paragraphs(6)
  )
end
```

Run ```$ rails db:seed``` and you'll see 100 hipster-themed blogposts in the Post database table and on the index page.

### Create the Search Form

```ruby
<%= form_tag(posts_path, method: "get") do %>
  <%= text_field_tag :search, params[:search], placeholder: "Enter search term" %>
  <%= submit_tag "Search" %>
<% end %>
```

### Add pg_search to the Post Model

```ruby
class Post < ApplicationRecord
  include PgSearch
  pg_search_scope :search_by_title_and_body, :against => [:title, :body],
    using: {
      :tsearch => {:prefix => true}
    }
end
```

### Filter the Search Params in the Controller

```ruby
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :edit, :update, :destroy]

  # GET /posts
  # GET /posts.json
  def index
    if params[:search]
      @search_results_posts = Post.search_by_title_and_body(params[:search])
    else
      @posts = Post.all
    end
  end
  etc...
end
  ```

### Add AJAX

Make the search form submit via AJAX

```ruby
<%= form_tag(posts_path, method: "get", remote: true) do %>
  <%= text_field_tag :search, params[:search], placeholder: "Enter search term" %>
  <%= submit_tag "Search" %>
<% end %>
```

Add a ```respond_to``` block so the index action can respond to the AJAX call with JavaScript.

```ruby
class PostsController < ApplicationController
  before_action :set_post, only: [:show, :edit, :update, :destroy]

  # GET /posts
  # GET /posts.json
  def index
    if params[:search]
      @search_results_posts = Post.search_by_title_and_body(params[:search])
      respond_to do |format|
        format.js { render partial: 'search-results'}
      end
    else
      @posts = Post.all
    end
  end
  ```

Wrap the blogpost table in a ```<div>``` with an ```id="blogpost-table"``` and right after the ```</div>``` add another ```<div>``` with an ```id="search-results"```:

```ruby
<div id="blogpost-table">
  <table>
    <thead>
      <tr>
        <th>Title</th>
        <th>Body</th>
        <th colspan="3"></th>
      </tr>
    </thead>

    <tbody>
      <% @posts.each do |post| %>
        <tr>
          <td><%= post.title %></td>
          <td><%= post.body %></td>
          <td><%= link_to 'Show', post %></td>
          <td><%= link_to 'Edit', edit_post_path(post) %></td>
          <td><%= link_to 'Destroy', post, method: :delete, data: { confirm: 'Are you sure?' } %></td>
        </tr>
      <% end %>
    </tbody>
  </table>
</div>
<div id="search-results">

</div>
```

In ```app/views/posts/``` create ```_search-results.js.rb```, a file that will store the jQuery that will hide the unfiltered blogpost-table div and display the search results in the search-results div. Add the following jQuery:

```javascript
$("#blogpost-table").hide();
$("#search-results").html("<%= escape_javascript(render :partial => 'results') %>");
```

The above code renders another partial called "results". It will hold the erb that will display the results of our search and gets injected into the search-results div. Create ```_results.html.erb``` in ```app/views/posts/```:

```ruby
<table>
  <thead>
    <tr>
      <th>Title</th>
      <th>Body</th>
      <th colspan="3"></th>
    </tr>
  </thead>

  <tbody>
    <% @search_results_posts.each.each do |post| %>
      <tr>
        <td><%= post.title %></td>
        <td><%= post.body %></td>
        <td><%= link_to 'Show', post %></td>
        <td><%= link_to 'Edit', edit_post_path(post) %></td>
        <td><%= link_to 'Destroy', post, method: :delete, data: { confirm: 'Are you sure?' } %></td>
      </tr>
    <% end %>
  </tbody>
</table>
```
