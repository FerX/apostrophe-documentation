---
title: 'Nunjucks helper functions: calling JavaScript functions from templates'
layout: tutorial
---

# Nunjucks helper functions: calling JavaScript functions from templates

In the ["Building Navigation" tutorial](building-navigation.md) we called `apos.pages.isAncestorOf` from our template code.

`isAncestorOf` is a "helper function," made available to templates by the `apostrophe-pages` module.

That's great, but what if we need to add our own helper function?

Let's take another look at the [link-widgets module we just created in the previous tutorial](custom-widgets.md).

Let's say we want to make the label optional, and use the URL as a label if no label is provided:

```javascript
// lib/modules/link-widgets/index.js, in our project
module.exports = {
  extend: 'apostrophe-widgets',
  label: 'Link to a Page',
  addFields: [
    {
      name: 'url',
      type: 'url',
      label: 'URL',
      required: true
    },
    {
      name: 'label',
      type: 'string',
      label: 'Label',
      required: false
    }
  ]
};
```

Now, in `views/widget.html`, we write:

```markup
{# lib/modules/link-widgets/views/widget.html, in our project #}
<h4>
  <a href="{{ data.widget.url }}">
    {{ data.widget.label or data.widget.url }}
  </a>
</h4>
```

> In Nunjucks templates, `and` and `or` are used, not `&&` and `||`.

But there's a problem: the URL is a bit clumsy-looking as a label, especially if there are links all over the page. No one needs to see `https://` over and over.

We could do string replacement in Nunjucks, but as a general rule, **the more logic you write in templates, the harder they are to maintain.** We should instead move that code up to a JavaScript "helper function" in our `link-widgets` module:

```javascript
module.exports = {
  // lib/modules/link-widgets/index.js, in our project
  extend: 'apostrophe-widgets',
  label: 'Link to a Page',
  alias: 'link',
  addFields: [
    {
      name: 'url',
      type: 'url',
      label: 'URL',
      required: true
    },
    {
      name: 'label',
      type: 'string',
      label: 'Label',
      required: false
    }
  ],
  construct: function(self, options) {
    self.addHelpers({
      stripHttp: function (s) {
        return s.replace(/^(https?:|)\/\//, '');
      }
    });
  }
};
```

Now, in `widget.html`, we can write:
```markup
{# lib/modules/link-widgets/views/widget.html, in our project #}
<h4>
  <a href="{{ data.widget.url }}">
    {{ data.widget.label or apos.link.stripHttp(data.widget.url) }}
  </a>
</h4>
```

**"What's going on in this code?"** In the JavaScript code for the module,
we start by adding an `alias` and set it to `link`. Without an alias,
it is clumsy to access the module's helpers from Nunjucks.

Next we add a `construct` function. This function runs once when the module
starts up. *This happens just once, not on every request.*

Inside `construct`, we call `self.addHelpers` to add some helper functions
that are visible to ounr Nunjucks templates. Any helper functions we add
here can be called from the template.

> **You must not call async functions, await promises, wait for
> callbacks, make network requests or database calls, etc. from
> inside a helper function.** Helpers must be synchronous, the
> value they return is the value you'll get.  Any attempt to use
> asynchronous code here will not work.
> 
> If you need to do asynchronous work to get data for your template,
> you should do it **before** the template runs. Write a
> [promise event handler](../../other/events.md), or a
> [widget `load` method](../../technical-overviews/how-apostrophe-handles-requests.md#widget-loaders).

## Passing data as a "helper"

In addition to functions, you can also pass data as helpers. This
can be helpful if, for example, you use the same options for
`apostrophe-rich-text-widgets` in many places. Just
pass the data object instead of a function.

For example, in your module you might create a `helpers` module just
for sharing helpers like this:

```javascript
// lib/modules/helpers/index.js in our project
// (Don't forget to enable this new module in `app.js`)
module.exports = {
  alias: 'helpers',
  construct: function(self, options) {
    self.addHelpers({
      toolbar: [ 'Styles' 'Bold', 'Italic', 'Link', 'Unlink' ]
    });
  }
};
```

```markup
{# In any template we can now write: #}
{{ apos.singleton(data.page, 'footer', 'apostrophe-rich-text',
  {
    toolbar: apos.helpers.toolbar
  }
) }}
```

## An alternative: Nunjucks filters

Helper functions are handy, but Nunjucks also has a "filter" syntax. For example, `upper` is a built-in Nunjucks filter:

```markup
<h1>{{ data.page.title | upper }}</h1>
```

You can find a [reference guide to ApostropheCMS nunjucks filters here](https://docs.apostrophecms.org/apostrophe/other/nunjucks-filters), and you can [learn about the standard Nunjucks filters here](https://mozilla.github.io/nunjucks/templating.html#builtin-filters). Both require no extra code on your part.

We can also add our own Nunjucks filters. Here's another version of our `index.js`:

```javascript
module.exports = {
  // lib/modules/link-widgets/index.js, in our project
  extend: 'apostrophe-widgets',
  label: 'Link to a Page',
  alias: 'link',
  addFields: [
    {
      name: 'url',
      type: 'url',
      label: 'URL',
      required: true
    },
    {
      name: 'label',
      type: 'string',
      label: 'Label',
      required: false
    }
  ],
  construct: function(self, options) {
    self.addFilterrs({
      stripHttp: function (s) {
        return s.replace(/^(https?:|)\/\//, '');
      }
    });
  }
};
```

Now, in `widget.html`, we can use the new filter:
```markup
{# lib/modules/link-widgets/views/widget.html, in our project #}
<h4>
  <a href="{{ data.widget.url }}">
    {{ data.widget.label or (data.widget.url | stripHttp) }}
  </a>
</h4>
```

Which technique should you use? If it's up to you. If your function is used a lot in your templates and takes just one argument, the `|` syntax can be convenient.

