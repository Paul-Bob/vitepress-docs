---
version: '1.0'
license: community
---

# Search

Finding what you're looking for fast is essential. That's why Avo leverages [ransack's](https://github.com/activerecord-hackery/ransack) powerful query language.

:::info
While we show you examples using `ransack`, you can use other search engines, so `ransack` is not mandatory.
:::

First, you need to add `ransack` as a dependency to your app (breaking change from Avo v1.10).

```ruby
# Gemfile
gem 'ransack'
```

## Enable search for a resource

To enable search for a resource, you need to add the `search_query` class variable to the resource file.

```ruby{3-5}
class UserResource < Avo::BaseResource
  self.title = :name
  self.search_query = -> do
    scope.ransack(id_eq: params[:q], first_name_cont: params[:q], last_name_cont: params[:q], m: "or").result(distinct: false)
  end

  # fields go here
end
```

The `search_query` block passes over the `params` object that holds the `q` param, the actual query string. It also provides the `scope` variable on which you run the query. That ensures that the [authorization scopes](./authorization.html#scopes) have been appropriately applied.

In this block, you may configure the search however strict or loose you need it. Check out [ransack's search matchers](https://github.com/activerecord-hackery/ransack#search-matchers) to compose the query better.

:::warning
If you're using ransack version 4 and up you must add `ransackable_attributes` and maybe more to your model in order for it to work. Read more about it [here](https://activerecord-hackery.github.io/ransack/going-further/other-notes/#authorization-allowlistingdenylisting).
:::

## Authorize search

<VersionReq version="2.29" />

Since Avo 2.29 search is authorized in policy files using the `search?` method.

```ruby
class UserPolicy < ApplicationPolicy
  def search?
    true
  end
end
```

If the `search?` method returns false, the search operation for that resource is not going to show up in the global search and the search box on index is not going to be displayed.

If you're using `search?` already in your policy file, you can alias it to some other method in you initializer using the `config.authorization_methods` config. More about that on [the authorization page](./authorization.html#using-different-policy-methods).

```ruby
Avo.configure do |config|
  config.authorization_methods = {
    search: 'avo_search?',
  }
  end
```

## Configure the search result

### Label

By default, the search results will be displayed as text. The text label will be the [title column](./resources.html#setting-the-title-of-the-resource) you previously configured.

<img :src="('/assets/img/search/search_blank.jpg')" alt="Blank search" class="border mb-4" />

You may configure that to be something more complex using the `as_label` option. That will take the final value of that field and display it as the label of the search result.

```ruby{9-11}
class PostResource < Avo::BaseResource
  self.title = :name
  self.search_query = -> do
    scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
  end

  field :id, as: :id
  field :name, as: :text, required: true, as_label: true
  field :complex_name, as: :text, hide_on: :all, as_label: true do |model|
    "[#{model.id}]#{model.name}"
  end
end
```

<img :src="('/assets/img/search/search_label.jpg')" alt="Search label" class="border mb-4" />

Notice the `hide_on: :all` option used to hide the computed `complex_name` attribute from the rest of the views. That is because you **may or may not** want to show that attribute in other views.

### Description

<div class="rounded-md bg-blue-50 p-4">
  <div class="flex">
    <div class="flex-shrink-0">
      <svg class="h-5 w-5 text-blue-400" xmlns="http://www.w3.org/2000/svg" viewBox="0 0 20 20" fill="currentColor">
        <path fill-rule="evenodd" d="M18 10a8 8 0 11-16 0 8 8 0 0116 0zm-7-4a1 1 0 11-2 0 1 1 0 012 0zM9 9a1 1 0 000 2v3a1 1 0 001 1h1a1 1 0 100-2v-3a1 1 0 00-1-1H9z" clip-rule="evenodd" />
      </svg>
    </div>
    <div class="ml-3 flex-1 md:flex md:justify-between">
      <div class="text-sm leading-5 text-blue-700">
        Search Description is a <a href="https://avohq.io/purchase/pro" target="_blank" class="underline">pro</a> feature
      </div>
    </div>
  </div>
</div>

You might want to show more than just the title in the search result. Avo provides the `as_description` option to add some more information.

```ruby{12-16}
class PostResource < Avo::BaseResource
  self.title = :name
  self.search_query = -> do
    scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
  end

  field :id, as: :id
  field :name, as: :text, required: true, as_label: true
  field :complex_name, as: :text, hide_on: :all, as_label: true do |model|
    "[#{model.id}]#{model.name}"
  end
  field :excerpt, as: :text, as_description: true do |model|
    ActionView::Base.full_sanitizer.sanitize(model.body).truncate 130
  rescue
    ""
  end
end
```

<img :src="('/assets/img/search/search_description.jpg')" alt="Search description" class="border mb-4" />

### Avatar

* Search Avatar is a [Pro feature](https://avohq.io/purchase/pro).

You may improve the results listing by adding an avatar to each search result. You do that by using the `as_avatar` attribute. This attribute has three options `:square`, `:rounded` or `:circle`. That influences the final roundness of the avatar.

```ruby{17}
class PostResource < Avo::BaseResource
  self.title = :name
  self.search_query = -> do
    scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
  end

  field :id, as: :id
  field :name, as: :text, required: true, as_label: true
  field :complex_name, as: :text, hide_on: :all, as_label: true do |model|
    "[#{model.id}]#{model.name}"
  end
  field :excerpt, as: :text, as_description: true do |model|
    ActionView::Base.full_sanitizer.sanitize(model.body).truncate 130
  rescue
    ""
  end
  field :cover_photo, as: :file, is_image: true, as_avatar: :rounded
end
```

<img :src="('/assets/img/search/search_avatar.jpg')" alt="Search avatar" class="border mb-4" />

### Header Help Text

You may improve the results listing header by adding a piece of text highlighting the fields you are looking for or any other instruction for the user. You do that by using the `search_query_help` attribute. This attribute takes a string and appends it to the title of the resource.

<img :src="('/assets/img/search/search_header_help.jpg')" alt="Search Header Help" class="border mb-4" />

```ruby{6}
class PostResource < Avo::BaseResource
  self.title = :name
  self.search_query = -> do
    scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
  end
  self.search_query_help = "- search by id"

  field :id, as: :id
end
```

## Resource search

When a resource has the `search_query` attribute, a new search input will be displayed on the `Index` view.

![](/assets/img/search/resource_search.jpg)

## Global search

<LicenseReq license="pro" />


Avo also has a global search feature. It will search through all the resources that have the `search_query` attribute present.

You open the global search input by clicking the trigger on the navbar or by using the <kbd>CMD</kbd> + <kbd>K</kbd> keyboard shortcut (<kbd>Ctrl</kbd> + <kbd>K</kbd> on windows).

<img :src="('/assets/img/search/global_search_trigger.jpg')" alt="Global search trigger" class="border mb-4" />

### Hide the global search

If you, by any chance, want to hide the global search, you can do so using this setting 👇

```ruby{3}
# config/initializers/avo.rb
Avo.configure do |config|
  config.disabled_features = [:global_search]
end
```

## Hide a resource from the global search

You might have a resource that you'd like to be able to perform a search on when on its `Index` page but not have it present in the global search. You can hide it using `hide_from_global_search = true`.

```ruby{8}
class TeamMembershipResource < Avo::BaseResource
  self.title = :id
  self.includes = [:user, :team]
  self.visible_on_sidebar = false
  self.search_query = -> do
    scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
  end
  self.hide_from_global_search = true

  field :id, as: :id
  field :user, as: :belongs_to
  field :team, as: :belongs_to
end
```

### Scope out global or resource searches

You may want to perform different searches on the `global` search from the `resource` search. You may use the `params[:global]` flag to figure that out.


```ruby{8}
class OrderResource < Avo::BaseResource
  self.search_query = -> do
    if params[:global]
      # Perform global search
      scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
    else
      # Perform resource search
      scope.ransack(id_eq: params[:q], details_cont: params[:q], m: "or").result(distinct: false)
    end
  end
end
```
