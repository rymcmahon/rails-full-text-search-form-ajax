### Rails Full-Text Search Form with AJAX

My most popular blogpost, with over 19,000 pageviews in just 18 months, is [Create a Simple Search Form with Rails](http://www.rymcmahon.com/articles/2). Nearly all of the traffic (92%) was from Google because the post consistently ranked in the top 3 Google search page results for the term "rails search form". I guess a lot of Rails devs are looking for tutorials on search forms.

I received many comments on the blogpost (some were even nice) and a few people asked how they could make the simple search form more sophisticated. In response to those requests, I thought I'd write an updated post that demonstrates how to build a complex, full-text search form that submits via AJAX for that smooth, modern app feel.

Let's create a simple blog app with a form that allows visitors to perform full-text searches of the blogpost titles and body text while rendering the results without a full-page refresh. I am using Rails 5.2, Ruby 2.4.1, and PostgreSQL 9.6.3 for this demo.

#### The Setup

Create a new Rails app with PostgreSQL as the database (Postgres is required for the full-text search):

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

Open ```app/assets/javascripts/application.js``` and add ```//= require jquery3``` so jQuery is available in the asset pipeline. The file should look like this:

```ruby
//= require rails-ujs
//= require jquery3
//= require activestorage
//= require turbolinks
//= require_tree .
```

### Seed the Database

Let's add some records to the database using the Faker gem. Go to ```app/db``` , open ```seeds.rb```, and add:

```ruby
100.times do
  Post.create(
    title: Faker::Hipster.sentence,
    body: Faker::Hipster.paragraphs(6)
  )
end
```

Run ```$ rails db:seed``` and you'll see 100 hipster-themed blogposts on the index page.

### Create the Search Form

Let's use a ```form_tag``` for the search form since we aren't saving data to the model.

```ruby
<%= form_tag(posts_path, method: "get") do %>
  <%= text_field_tag :search, params[:search], placeholder: "Enter search term" %>
  <%= submit_tag "Search" %>
<% end %>
```

### Add pg_search to the Post Model

First, let's define the difference between a simple search and a full-text search. Let's say we have a recipe app that allows users to search for recipes by their name and one of the most popular recipes is "Penne with Arrabiata". With a simple search, a phrase such as "penne arrabiata" returns zero matches because the search phrase did not include the word "with." With full-text search, however, searching for "penne arrabiata" will return the "Penne with Arrabiata" recipe and all other recipes with either of those words in the title. Full-text search is more useful and it's the kind of search visitors are expecting from modern apps.

Let's include the pg_search module in the model we want to search. After it's included, create a scope and choose the attributes you want the search to use to look for matches (in our case, ```:title``` and ```:body```. Setting ```:tsearch => {:prefix => true} ``` will give us the full-text search we desire and it will allow searches for partial words, so a search for "pen" will return "Penne with Arrabiata".

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

In the controller, let’s create a conditional that displays the search results when the search form is submitted or it displays all of the blogposts in all other circumstances.

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

Submitting a form with AJAX allows us to insert search results into the DOM without reloading the entire page. This is a nice UX touch for users who may perform multiple searches during their visit. To submit a form via AJAX in Rails, add ```remote: true``` to the ```form_tag``` arguments.

```ruby
<%= form_tag(posts_path, method: "get", remote: true) do %>
  <%= text_field_tag :search, params[:search], placeholder: "Enter search term" %>
  <%= submit_tag "Search" %>
<% end %>
```

Next, we need a ```respond_to``` block so the index action can respond to the AJAX call with JavaScript, since we are no longer submitting the form via HTML. The ```respond_to``` block is going to render a partial called "search-results" that we will create shortly.

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

On the index page, we need to wrap a ```<div>``` around the table and assign it an id of "blogpost-table". We also need to place an empty ```<div>``` with an id of "search-results" immediately after the closing tag of the first ```<div>```. Using jQuery, we are going to hide the "blogpost-table" ```<div>``` when a search is performed and insert the results in the "search-results" ```<div>``` via JavaScript.

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

### Sprinkles of jQuery
In ```app/views/posts/``` create ```_search-results.js.rb```, a file that will store the jQuery that will hide the unfiltered blogpost-table ```<div>``` and display the search results in the search-results ```<div>```. Add the following jQuery:

```javascript
$("#blogpost-table").hide();
$("#search-results").html("<%= escape_javascript(render :partial => 'results') %>");
```

The above code renders another partial called "results". It will hold the ERB that will display the results of our search and gets injected into ```<div id="search-results">.``` Create ```_results.html.erb``` in ```app/views/posts/``` and add the same code for the table in ```views/posts/index.html.erb```, but be sure to change ```@posts``` to  ```@search_results_posts``` inside the loop:

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
    <% @search_results_posts.each do |post| %>
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

That's it! Try searching for random hipster phrases and watch the matching blogposts appear without a full-page reload! The pg_search gem has a bunch of other options you can add and you can enhance this feature even further by adding autocomplete or a site-wide search that searches multiple models (maybe that’s v3 of this post?)

Check out a live demo of the full-text AJAX search form at [https://rails-full-text-search-form.herokuapp.com/posts](https://rails-full-text-search-form.herokuapp.com/posts)
