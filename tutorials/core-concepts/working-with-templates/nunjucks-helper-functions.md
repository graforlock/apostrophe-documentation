---
title: 'Nunjucks helper functions: calling JavaScript functions from templates'
layout: tutorial
---

# Nunjucks helper functions: calling JavaScript functions from templates

In the ["Building Navigation" tutorial](/tutorials/core-concepts/pages-and-navigation/connecting-pages.md) you called `apos.pages.isAncestorOf` from your template code.

`isAncestorOf` is a "helper function," made available to templates by the `apostrophe-pages` module.

That's great, but what if you need to add your own helper function?

Let's take another look at the [link-widgets module you just created in the previous tutorial](/tutorials/core-concepts/editable-content-on-pages/custom-widgets.md).

Let's say you want to make the label optional, and use the URL as a label if no label is provided:

{% code-tabs %}
{% code-tabs-item title="lib/modules/link-widgets/index.js" %}
```javascript
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
{% endcode-tabs-item %}
{% endcode-tabs %}

Now, in `views/widget.html`, write:

{% code-tabs %}
{% code-tabs-item title="views/widget.html" %}
```markup
<h4>
  <a href="{{ data.widget.url }}">
    {{ data.widget.label or data.widget.url }}
  </a>
</h4>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

**In Nunjucks templates, `and` and `or` are used, not `&&` and `||`.**

But there's a problem: the URL is a bit clumsy-looking as a label, especially if there are links all over the page. No one needs to see `https://` over and over.

You could do string replacement in Nunjucks, but as a general rule, **the more logic you write in templates, the harder they are to maintain.** You should instead move that code up to a JavaScript "helper function" in your `link-widgets` module:

{% code-tabs %}
{% code-tabs-item title="lib/modules/link-widgets/index.js" %}
```javascript
module.exports = {
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
{% endcode-tabs-item %}
{% endcode-tabs %}

Now, in `widget.html`, write:

{% code-tabs %}
{% code-tabs-item title="views/widget.html" %}
```markup
<h4>
  <a href="{{ data.widget.url }}">
    {{ data.widget.label or apos.link.stripHttp(data.widget.url) }}
  </a>
</h4>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

**"What's going on in this code?"** In the JavaScript code for the module, we start by adding an `alias` and set it to `link`. Without an alias, it is clumsy to access the module's helpers from Nunjucks.

Next we add a `construct` function. This function runs once when the module starts up. *This happens just once, not on every request.*

Inside `construct`, we call `self.addHelpers` to add some helper functions that are visible to your Nunjucks templates. Any helper functions we add here can be called from the template.

{% hint style='info' %}
**You must not call async functions, await promises, wait for callbacks, make network requests or database calls, etc. from inside a helper function.** Helpers must be synchronous, the value they return is the value you'll get.  Any attempt to use asynchronous code here, or anywhere in a template, **Will not work**.

If you need to do asynchronous work to get data for your template, you should do it **before** the template runs. Write a [promise event handler](/tutorials/advanced-development/promise-events/promise-events.md), or a [widget `load` method](/tutorials/advanced-development/how-apostrophe-handles-requests.md#widget-loaders).
{% endhint %}

## Returning markup from a helper

Normally, the output of helper functions is automatically escaped by Nunjucks. That is, if your function returns `<h1>`, that will become `&lt;h1&gt;`, rendering the characters literally. This is a built-in Nunjucks feature to protect against [CSRF attacks](https://www.owasp.org/index.php/Cross-Site_Request_Forgery).

If your helper function returns markup, and you know that markup is safe (you have eliminated or escaped any user input that could generate HTML tags), you can return a "safe string" like this:

```javascript
self.addHelpers({
  title: function(page) {
    return self.apos.templates.safe('<h1>' + self.apos.utils.escapeHtml(page.title) + '</h1>');
  }
});
```

## Passing data as a "helper"

In addition to functions, you can also pass data as helpers. This can be helpful if, for example, you use the same options for `apostrophe-rich-text-widgets` in many places. Just pass the data object instead of a function.

For example, in your module you might create a `helpers` module just for sharing helpers like this:

{% code-tabs %}
{% code-tabs-item title="lib/modules/helpers/index.js" %}
```javascript
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
{% endcode-tabs-item %}
{% endcode-tabs %}


```markup
{# In any template you can now write: #}
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

You can also add your own Nunjucks filters. Here's another version of `index.js`:

{% code-tabs %}
{% code-tabs-item title="lib/modules/link-widgets/index.js" %}
```javascript
module.exports = {
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
{% endcode-tabs-item %}
{% endcode-tabs %}

Now, in `widget.html`, use the new filter:

{% code-tabs %}
{% code-tabs-item title="lib/modules/link-widgets/views/widget.html" %}
```markup
<h4>
  <a href="{{ data.widget.url }}">
    {{ data.widget.label or (data.widget.url | stripHttp) }}
  </a>
</h4>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

Which technique should you use? If it's up to you. If your function is used a lot in your templates and takes just one argument, the `|` syntax can be convenient.

