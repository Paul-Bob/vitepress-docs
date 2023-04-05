# Upgrade guide

We generally push changes behind the scenes, so you don't have to update your code, but sometimes the public API is updated too.

Follow these guides to make sure your configuration files are up to date.

## Upgrade from 2.29 to 2.30

We have discovered that assigning different weights to two methods that police the same thing can be a complex and challenging task. As a result `upload_attachments?`, `download_attachments?`, and `delete_attachments?` become obsolete.
You can safely delete it from all policies.

Go to [attachments policies methods](./../2.0/authorization.html#attachments) to apply the new methods on your policies.

## Upgrade from 2.28 to 2.29

### Add the `search?` method to your policies

Policies are now authorized against the `search?` method in your Policy files. So for each policy file you're using add it and return true if you want the user to see the search box or to be able to execute a search query.

```ruby
class UserPolicy < ApplicationPolicy
  def search?
    true
  end
end
```

You can alias it to some other method in you initializer using the `config.authorization_methods` config. More about that on [the authorization page](./authorization.html#using-different-policy-methods).

```ruby
Avo.configure do |config|
  config.authorization_methods = {
    search: 'avo_search?',
  }
  end
```

## Upgrade from 2.26 to 2.27

### Replace `resolve_find_scope` with `find_record_method`

In 2.27 we pushed a long awaited feature where we Avo supports much better gems like `friendly` and `prefixed_ids`.

Instead of using `self.resolve_find_scope` (which will be deprecated in the future), please use `find_record_method` which will give you more control on how to find records within Avo.

```ruby
# Before
self.resolve_find_scope = ->(model_class:) do
  model_class.friendly
end

# After
self.find_record_method = ->(model_class:, id:, params:) do
  model_class.friendly.find id
end
```

## Upgrade from 2.20 to 2.21

In version 2.20 we discovered a bug in the [associated policy methods](./authorization#associations) (`create_{association}?`, `attach_{association}?`, etc.).
The record that you would get in that method would have been the parent record, not the actual record that you were trying to authorize.
After further investigation we found out that we can sometimes expose the parent record and sometimes expose the child record.

In the `Post` `has_many` `Comments` example, when you want to authorize `show_comments?` in `PostPolicy` you will have a `Comment` instance as the `record` variable, but when you try to authorize the `attach_comments?`, you won't have that `Comment` instance because you want to create one, but we expose the parent `Post` instance so you have more information about that authorization action that you're trying to make.

So there isn't a guide to follow per-se for this upgrade, but you just have to check your association policy methods are applied correctly.

### Updated a few translation strings

We found some incoherences around I18n localization on empty tables messages. More info on that in [this discussion](https://github.com/avo-hq/avo/issues/1487).
So we decided to keep it simple, changing those messages to a generic message that applies to all tables.

```ruby
# Before
no_item_found: No %{item} found // [!code focus]
no_related_item_found: No related %{item} found // [!code focus]

# After
no_item_found: No record found // [!code focus]
no_related_item_found: No related record found // [!code focus]
```

To fix that run `bin/rails generate avo:locales` to re-generate the locale files.

## Upgrade from 2.19 to 2.20

If you have some action declared inside `self.show_controls = -> do` block, you should assure that action it's also declared on the host resource, outside of that block. That happens because we added `arguments` on actions and in order to get the action arguments we search inside resource declared actions. We already noticed that arguments declared inside `self.show_controls = -> do` are not respected and we are improving this whole experience on Avo 3.0.

```ruby
class FishResource < Avo::BaseResource
  self.title = :name

  self.show_controls = -> do
    # In order to be used here
    action ReleaseFish, style: :primary, color: :fuchsia
  end

  # Should be declared here
  action ReleaseFish, arguments: { both_actions: "Will use them" }
end

```


## Upgrade from 2.18 to 2.19

### Add a `dashboards` directory

If you happen to get a `Zeitwerk::Error` regarding `/app/avo/dashboards is not a directory`, just create a `dashboards` directory inside `/app/avo`. Run `mkdir app/avo/dashboards` in the root path of your project.

We're not sure why this error pops up, and it doesn't happen to everyone.

### Remove the params from the `visible` block in actions

In 2.19, we added the visibility block for filters. We used the same logic applyed to the actions visible block and in that upgrade process, we changed the way that visibility block works, you need to do a minor update to your code and remove the arguments from yours visibility blocks inside your actions. This way, both, filter and actions, uses the same visibility block and it's more flexible and future-proof. From now on we can give access to new params inside that blocks without making you to change your code again.

```ruby
# Before
self.visible = ->(resource:, view:) do // [!code focus]
  true
end

# After
self.visible = -> do // [!code focus]
  true
end
```

To make it easier for you to migrate, we made this ruby script
```ruby
DONT_TOUCH = ['.', '..', $0]
OLD_VISIBLE_BLOCK = "self.visible = ->(resource:, view:) do"
NEW_VISIBLE_BLOCK = "self.visible = -> do"

def remove_args_from_visible_block(file_name)
  content = File.read file_name
  content.gsub!(OLD_VISIBLE_BLOCK, NEW_VISIBLE_BLOCK)

  File.open(file_name, "w") { |file| file << content }
end

Dir.foreach(".") {|file_name| remove_args_from_visible_block file_name unless DONT_TOUCH.include? file_name}
```

**Usage**
- Create a ruby file in your **actions folder** (ex: `app/avo/actions/remove_args_from_visible_block.rb`) with the content above.
- Open terminal inside your **actions folder**
- Execute the script: `$ ruby remove_args_from_visible_block.rb`
- Remove the upgrade script

**Expected behavior**
The script should replace all `self.visible = ->(resource:, view:) do` with `self.visible = -> do`.


## Upgrade from 2.17 to 2.18

### Manually require some gems

In an effort to slim Avo down and require less gems by default, we removed some default dependencies.

:::warning Action required
- If you're using using the `file` and `files` fields include `activestorage` and `image_processing` gems
- If you're using using the `country` field include the `countries` gem
- If you're using using the [Dashboards](./dashboards) feature include the `chartkick` gem
- If you're using using the [Authorization](./authorization) feature include the `pundit` gem
:::

To successfully migrate, your `Gemfile` should contain these gems:

```ruby
# Minimal authorization through OO design and pure Ruby classes
gem "pundit"

# Active Storage makes it simple to upload and reference files
gem "activestorage"

# High-level image processing wrapper for libvips and ImageMagick/GraphicsMagick
gem "image_processing"

# All sorts of useful information about every country packaged as convenient little country objects.
gem "countries"

# Create beautiful JavaScript charts with one line of Ruby
gem "chartkick"
```

:::info No action required
- Removed the `rails` requirement in favor of `activerecord` and `actionview`.
:::

## Upgrade from 2.16 to 2.17

### Field internals changes

:::warning
You only need to follow these instructions if you have added custom fields to Avo. If you haven't, you'll probably be just fine without
:::

We changed the way we handle the field and field wrapper internals.

1. Merged `edit_field_wrapper` and `show_field_wrapper` and their components into one common `field_wrapper` method and component combo. The `index_field_wrapper` remains the same.
1. `displayed_in_modal` was renamed to `compact`
1. Added the `stacked` option to the <Show /> and <Edit /> fields and field wrapper that makes the label and field have a column-like appearance like they do on mobile.
1. Added a `field_wrapper_args` method to the `Avo::Fields::EditComponent` (to the base fields for <Show /> and <Edit /> fields).

These changes will not affect you at all if you don't have any custom fields added, but if you do, you will need to go into your custom fields and tweak the <Edit /> and <Show /> partials for your custom fields like so:

```diff
- <%= edit_field_wrapper field: @field, index: @index, form: @form, resource: @resource, displayed_in_modal: @displayed_in_modal do %>
+ <%= field_wrapper **field_wrapper_args do %>
```

That will ensure the right arguments are going to be passed to the field wraper component. You may, however, tweak them like so:

```erb
<%= field_wrapper **field_wrapper_args, compact: true do %>
```

Also we recommend using methods instead of instance variables for `field`, `form`, `resource`, etc.

## Upgrade from 2.13 to 2.14

Please ensure that you have `Rails.application.secrets.secret_key_base` or `ENV['SECRET_KEY_BASE']` available and at least 32 characters long.

Also, run `bin/rails generate avo:locales` to regenerate the locales files.

## Upgrade from 2.12 to 2.13

### Remove the params from the `search_query` block

In 2.13, we added scoped search on `has_many` associations. Unfortunately, in that upgrade process, we changed the way `search_query` works, and you need to do a minor update to your code and remove the block params. This way, it's more flexible and future-proof.

```ruby
# Before
self.search_query = ->(params:) do
  scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
end

# After
self.search_query = -> do
  scope.ransack(id_eq: params[:q], m: "or").result(distinct: false)
end
```

To make it easier for you to migrate, we made this ruby script
```ruby
DONT_TOUCH = ['.', '..', $0]
OLD_SEARCH_QUERY = "self.search_query = ->(params:) do"
NEW_SEARCH_QUERY = "self.search_query = -> do"

def remove_params_keyword(file_name)
  content = File.read file_name
  content.gsub!(OLD_SEARCH_QUERY, NEW_SEARCH_QUERY)

  File.open(file_name, "w") { |file| file << content }
end

Dir.foreach(".") {|file_name| remove_params_keyword file_name unless DONT_TOUCH.include? file_name}
```

**Usage**
- Create a ruby file in your **resources folder** (ex: `app/avo/resources/remove_params_keyword.rb`) with the content above.
- Execute the script: `$ ruby remove_params_keyword.rb`
- Remove the upgrade script

**Expected behavior**
The script should replace all `self.search_query = ->(params:) do` with `self.search_query = -> do`.

## Upgrade from 2.10 to 2.11

### Avo uses the `locale` configuration from the initializer

In 2.11 a change was pushed, so Avo uses the `locale` configuration option from the `avo.rb` initializer.

```ruby{2}
Avo.configure do |config|
  config.locale = :en # default is nil
end
```

So if you get locale-related crashes after an update, make sure the locale is set to a valid locale or set it to `nil` if you want to fall back to what you have configured in your app.

### Change the `format` option in the date time and date fields

One of the features of the date-_ime field is to show the value in the browser's time zone. However, we can't know that until we load the page, hence we are going to parse and format the date on the browser side.

Avo uses luxon to do that, so you'll have to change the tokens to match. Use this list of tokens [here](https://moment.github.io/luxon/#/formatting?id=table-of-tokens).

## Upgrade from 2.8 to 2.9

### Avo generates paths based on the resource name not the model name

We made this change so you get more predictable paths. You really shouldn't have to do anything. You will be affected only if you have hardcoded paths towards Avo.
For example, if you have a `SubscriptionResource` with `self.model_class = Pay::Subscription`, the resource path before was `/avo/resurces/pay_subscriptions` and now it's going to be `/avo/resurces/subscriptions`. the path will be build following the resource class, not the model class.

Get more information on [this ticket](https://github.com/avo-hq/avo/pull/953).

### Actions are visible on the Edit view

We changed the way we display [Actions](actions). Now they will be visible on the `Edit` view too. You my disable that using the `visible` option on each action.

```ruby
self.visible = -> (resource:, view:) { view.in?([:index, :show]) }
```

### Eject translation files

Before 2.9 [we were loading](https://github.com/avo-hq/avo/pull/960/files#diff-3d269fbd54784c3eeb51983191c4565230a00b86e352c58b493282c916a18018L20) the translation directory in the engine file so your translation files would get less important and wouldn't load for those languages we provided. We fixed that by removing Avo's directory

To fix this, eject the files we provide out of the box using `bin/rails generate avo:locales`.

## Upgrade from 2.7 to 2.8

### Pass the `resource` to the `show_field_wrapper` in custom fields

When we added the Stimulus JS integration we used the field wrappers to add the HTML attributes. The fields generator didn't pass the `resource` to the field wrapper so all fields generated before don't have it and you need to add it manually. `resource: @resource, ` to `show_field_wrapper`.

```erb{2,7}
<!-- Before -->
<%= show_field_wrapper field: @field, index: @index do %>
  <%= @field.value %>
<% end %>

<!-- After -->
<%= show_field_wrapper field: @field, resource: @resource, index: @index do %>
  <%= @field.value %>
<% end %>
```

### Changed the way the `ranges` option is processed

We no longer process the `ranges` option to cast integers as days. The `ranges` option is passed to the [`options_for_select`](https://apidock.com/rails/v5.2.3/ActionView/Helpers/FormOptionsHelper/options_for_select) helper, so it behaves more like a regular `select_tag`.

```ruby{6,13-23}
# Before
class UsersMetric < Avo::Dashboards::MetricCard
  self.id = 'users_metric'
  self.label = 'Users count'
  self.initial_range = 30
  self.ranges = [7, 30, 60, 365, 'TODAY', 'MTD', 'QTD', 'YTD', 'ALL']
end

# After
class UsersMetric < Avo::Dashboards::MetricCard
  self.id = 'users_metric'
  self.label = 'Users count'
  self.initial_range = 30
  self.ranges = {
    "7 days": 7,
    "30 days": 30,
    "60 days": 60,
    "365 days": 365,
    Today: "TODAY",
    "Month to date": "MTD",
    "Quarter to date": "QTD",
    "Year to date": "YTD",
    All: "ALL"
  }
end
```

## Upgrade from 2.5 to 2.6

### Change the way the cards run their queries

We made a change to the way you build your queries in cards. Instead of using the `query` block, you can use the query method.

The change should be straightforward and shouldn't really impact the logic of your card. You'll have access to all the same data as before.

```ruby{11-14,16-19}
class AmountRaised < Avo::Dashboards::MetricCard
  self.id = "amount_raised"
  self.label = "Amount raised"
  # self.description = "Some description"
  # self.cols = 1
  # self.initial_range = 30
  # self.prefix = ""
  # self.suffix = ""

  # Before
  query do
    result 9001
  end

  # Current
  def query
    result 9001
  end
end
```

## Upgrade from 2.4 to 2.5

### Change the way the scope is declared in associations

We changed how we add scopes to associations to make the API more flexible and extendable. You have to append `query.` to the scope.

Also, you now have access to a few more pieces of information inside that block. You can use the `parent`, which is the actual parent record (`User` in the example below) of that association.

```ruby{16,22}
# app/models/comment.rb
class Comment < ApplicationRecord
  belongs_to :user, optional: true

  scope :starts_with, -> (prefix) { where('LOWER(body) LIKE ?', "#{prefix}%") }
end

# app/models/user.rb
class User < ApplicationRecord
  has_many :comments
end

# app/avo/resources/user_resource.rb
class UserResource < Avo::BaseResource
  # Version before v2.5.0
  field :comments, as: :has_many, scope: -> { starts_with :a }
end

# app/avo/resources/user_resource.rb
class UserResource < Avo::BaseResource
  # Version after v2.5.0
  field :comments, as: :has_many, scope: -> { query.starts_with :a }
end
```

## Upgrade from 1.x to 2.0

### Update the gem

Run `bundle update avo` to update your gem. If you have a Pro license, follow [this guide](https://docs.avohq.io/2.0/licensing.html#upgrade-your-1-0-license-to-2-0) to update your license.

### Update your sidebar & profile partials

We changed some of the remaining partials to `view_component`s.

### View components

Renamed the following view components:

- `NavigationLinkComponent` to `SidebarItemComponent`.
- `NavigationHeadingComponent` to `SidebarHeadingComponent`.

### Translations

We added the following tags:

 - `avo.details`

Removed the following tags:

- `avo.resource_details`
- `avo.update_item`

### Controllers

Renamed `RelationsController` to `AssociationsController`
