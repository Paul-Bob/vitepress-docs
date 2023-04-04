---
version: '1.0'
license: community
---

:::warning
It's important to set the `inverse_of` as often as possible to your model's association attribute.
:::

# Has One

The `HasOne` association shows the unfolded view of your `has_one` association. It's like peaking on the `Show` view of that associated record. The user can also access the `Attach` and `Detach` buttons.

```ruby
field :admin, as: :has_one
```

<img :src="('/assets/img/associations/has-one.jpg')" alt="Has one" class="border mb-4" />

## Options

<!-- @include: ./../common/associations_searchable_option_common.md-->
<!-- @include: ./../common/associations_attach_scope_option_common.md-->

<!-- @include: ./../common/show_on_edit_common.md-->
