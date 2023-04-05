---
license: community
feedbackId: 837
---

# Actions

Avo actions allow you to perform specific tasks on one or more of your records.

For example, you might want to mark a user as active/inactive and optionally send a message that may be customized by the person that wants to run the action.

Once you attach an action to a resource using the `action` method, it will appear in the **Actions** dropdown. By default, actions appear on the `Index`, `Show`, and `Edit` views. Versions previous to 2.9 would only display the actions on the `Index` and `Show` views.

![Actions dropdown](/assets/img/actions/actions-dropdown.gif)

:::info
Since version <Version version="2.13" /> you may use the [customizable controls](./customizable-controls) feature to show the actions outside the dropdown.
:::

## Overview

You generate one running `bin/rails generate avo:action toggle_active`, creating an action configuration file.

```ruby
class ToggleInactive < Avo::BaseAction
  self.name = 'Toggle inactive'

  field :notify_user, as: :boolean, default: true
  field :message, as: :text, default: 'Your account has been marked as inactive.'

  def handle(**args)
    models, fields, current_user, resource = args.values_at(:models, :fields, :current_user, :resource)

    models.each do |model|
      if model.active
        model.update active: false
      else
        model.update active: true
      end

      # Optionally, you may send a notification with the message to that user from inside the action
      UserMailer.with(user: model).toggle_active(fields["message"]).deliver_later
    end

    succeed 'Perfect!'
  end
end
```

You may add fields to the action just as you do it in a resource. Adding fields is optional. You may have actions that don't have any fields attached.

```ruby
field :notify_user, as: :boolean
field :message, as: :textarea, default: 'Your account has been marked as inactive.'
```

:::warning Files authorization
If you're using the `file` field on an action and attach it to a resource that's using the authorization feature, please ensure you have the `upload_{field_id}?` policy method returning `true`. Otherwise, the `file` input might be hidden.

More about this on the [authorization page](./authorization#attachments).
:::


![Actions](/assets/img/actions/action-fields.jpg)

The `handle` method is where the magic happens. That is where you put your action logic. In this method, you will have access to the selected `models` (if there's only one, it will be automatically wrapped in an array) and the values passed to the `fields`.

```ruby
def handle(**args)
  models, fields = args.values_at(:models, :fields)

  models.each do |model|
    if model.active
      model.update active: false
    else
      model.update active: true
    end

    # Optionally, you may send a notification with the message to that user.
    UserMailer.with(user: model).toggle_active(fields["message"]).deliver_later
  end

  succeed 'Perfect!'
end
```

## Registering actions

To add an action to one of your resources, you need to declare it on the resource using the `action` method.

```ruby{8}
class UserResource < Avo::BaseResource
  self.title = :name
  self.search = [:id, :first_name, :last_name]

  field :id, as: :id
  # other fields

  action ToggleActive
end
```

## Action responses

After an action runs, you may use several methods to respond to the user. For example, you may respond with just a message or with a message and an action.

The default response is to reload the page and show the _Action ran successfully_ message.

### Message responses

You will have four message response methods at your disposal `succeed`, `error`, `warn`, and `inform`. These will render the user green, red, orange, and blue alerts.

```ruby{4-7}
def handle(**args)
  # Demo handle action

  succeed "Success response ✌️"
  warn "Warning response ✌️"
  inform "Info response ✌️"
  error "Error response ✌️"
end
```

:::warning
Since Avo 2.20 we deprecated the `fail` method in favor of `error`.
:::

<img :src="('/assets/img/actions/alert-responses.png')" alt="Avo alert responses" class="border inline-block" />

### Run actions silently

You may want to run an action and show no notification when it's done. That is useful for redirect scenarios. You can use the `silent` response for that.

```ruby
def handle(**args)
  # Demo handle action

  redirect_to "/admin/some-tool"
  silent
end
```

## Response types

After you notify the user about what happened through a message, you may want to execute an action like `reload` (default action) or `redirect_to`. You may use message and action responses together.

```ruby{14}
def handle(**args)
  models = args[:models]

  models.each do |model|
    if model.admin?
      error "Can't mark inactive! The user is an admin."
    else
      model.update active: false

      succeed "Done! User marked as inactive!"
    end
  end

  reload
end
```

The available action responses are:

### `reload`

When you use `reload`, a full-page reload will be triggered.

```ruby{9}
def handle(**args)
  models = args[:models]

  models.each do |project|
    project.update active: false
  end

  succeed 'Done!'
  reload
end
```

### `redirect_to`

`redirect_to` will execute a redirect to a new path of your app.

```ruby{9}
def handle(**args)
  models = args[:models]

  models.each do |project|
    project.update active: false
  end

  succeed 'Done!'
  redirect_to avo.resources_users_path
end
```

### `download`

`download` will start a file download to your specified `path` and `filename`.

**You need to set may_download_file to true for the download response to work like below**. That's required because we can't respond with a file download (send_data) when making a Turbo request.

If you find another way, please let us know 😅.

::: code-group

```ruby{3,19} [app/avo/actions/download_file.rb]
class DownloadFile < Avo::BaseAction
  self.name = "Download file"
  self.may_download_file = true

  def handle(**args)
    models = args[:models]

    filename = "projects.csv"
    report_data = []

    models.each do |project|
      report_data << project.generate_report_data
    end

    succeed 'Done!'

    if report_data.present? and filename.present?
      download report_data, filename
    end
  end
end
```

```ruby{5} [app/avo/resources/project_resource.rb]
class ProjectResource < Avo::BaseResource

  # fields here

  action DownloadFile
end
```
:::

### `keep_modal_open`

There might be situations where you want to run an action and if it fails, respond back to the user with some feedback but still keep it open and the inputs filled in.

`keep_modal_open` will tell Avo to keep the modal open.

```ruby
class KeepModalOpenAction < Avo::BaseAction
  self.name = "Keep Modal Open"
  self.standalone = true

  field :name, as: :text
  field :birthday, as: :date

  def handle(**args)
    begin
    user = User.create args[:fields]
    rescue => error
      error "Something happened: #{error.message}"
      keep_modal_open
      return
    end

    succeed "All good ✌️"
  end
end
## Customization

```ruby{2-6}
class TogglePublished < Avo::BaseAction
  self.name = 'Mark inactive'
  self.message = 'Are you sure you want to mark this user as inactive?'
  self.confirm_button_label = 'Mark inactive'
  self.cancel_button_label = 'Not yet'
  self.no_confirmation = true
```

### Customize the message

You may update the `self.message` class attribute to customize the message if there are no fields present.

#### Callable message

<VersionReq version="2.21" />

Since version `2.21` you can pass a block to `self.message` where you have access to a baunch of variables.

```ruby
class ReleaseFish < Avo::BaseAction
  self.message = -> {
    # you have access to:
    # - params
    # - current_user
    # - context
    # - view_context
    # - request
    # - resource
    # - record
    "Are you sure you want to release the #{record.name}?"
  }
end
```

<!-- <img :src="('/assets/img/actions/actions-message.jpg')" alt="Avo message" class="border mb-4" /> -->

### Customize the buttons

You may customize the labels for the action buttons using `confirm_button_label` and `cancel_button_label`.

<img :src="('/assets/img/actions/actions-button-labels.jpg')" alt="Avo button labels" class="border mb-4" />

### No confirmation actions

You will be prompted by a confirmation modal when you run an action. If you don't want to show the confirmation modal, pass in the `self.no_confirmation = true` class attribute. That will execute the action without showing the modal at all.

## Standalone actions

You may need to run actions that are not necessarily tied to a model. Standalone actions help you do just that. Add `self.standalone` to an existing action or generate a new one using the `--standalone` option (`bin/rails generate avo:action global_action --standalone`).

```ruby{3}
class DummyAction < Avo::BaseAction
  self.name = "Dummy action"
  self.standalone = true

  def handle(**args)
    fields, current_user, resource = args.values_at(:fields, :current_user, :resource)

    # Do something here

    succeed 'Yup'
  end
end
```

## Actions visibility

You may want to hide specific actions on screens, like a standalone action on the `Show` screen. You can do that using the `self.visible` attribute.

```ruby{4}
class DummyAction < Avo::BaseAction
  self.name = "Dummy action"
  self.standalone = true
  self.visible = -> { view == :index }

  def handle(**args)
    fields, current_user, resource = args.values_at(:fields, :current_user, :resource)

    # Do something here

    succeed 'Yup'
  end
end
```

By default, actions are visible on the `Index`, `Show`, and `Edit` views, but you can enable them on the `New` screen, too (from version 2.9.0).

```ruby
self.visible = -> { view == :new }

# Or use this if you want them to be visible on any view
self.visible = -> { true }
```

Inside the visible block you can acces the following variables:
```ruby
  self.visible = -> do
    #   You have access to:
    #   block
    #   context
    #   current_user
    #   params
    #   parent_model
    #   parent_resource
    #   resource
    #   view
    #   view_context
  end
```

## Actions authorization

:::warning
Using the Pundit policies, you can restrict access to actions using the `act_on?` method. If you think you should see an action on a resource and you don't, please check the policy method.

More info [here](./authorization#act-on)
:::

## Actions arguments

Actions can have different behaviors according to their host resource. In order to achieve that, arguments must be passed like on the example below:

```ruby{9-11}
class FishResource < Avo::BaseResource
  self.title = :name

  field :id, as: :id
  field :name, as: :text
  field :user, as: :belongs_to
  field :type, as: :text, hide_on: :forms

  action DummyAction, arguments: {
    special_message: true
  }
end
```

Now, the arguments can be accessed inside `DummyAction` ***`handle` method*** and on the ***`visible` block***!

```ruby{4-6,8-14}
class DummyAction < Avo::BaseAction
  self.name = "Dummy action"
  self.standalone = true
  self.visible = -> do
    arguments[:special_message]
  end

  def handle(**args)
    if arguments[:special_message]
      succeed "I love 🥑"
    else
      succeed "Success response ✌️"
    end
  end
end
```
