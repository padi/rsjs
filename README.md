# rsjs

<!-- {h1:.massive-header.-with-tagline} -->

> Reasonable System for JavaScript Structure

:construction: This document is a work in progress. Please feel free to contest any of the points raised here in [the issues](https://github.com/rstacruz/rsjs/issues). Also see [rscss], a document on CSS conventions that follows a similar line of thinking.

## Problem

For a typical non-[SPA] website, it will eventually be apparent that there needs to be conventions enforced for consistency. While [styleguides][abnb] cover coding styles, conventions on namespacing and file structures are usually absent.

[abnb]: https://github.com/airbnb/javascript
[SPA]: https://en.wikipedia.org/wiki/Single-page_application

### The jQuery soup anti-pattern
You will typically see Rails projects with behaviors randomly attached to classes, such as the problematic example below.

```html
<script src='blogpost.js'></script>
<div class='author footnote'>
  This article was written by
  <a class='profile-link' href="/user/rstacruz">Rico Sta. Cruz</a>.
</div>
```

```js
$(function () {
  $('.author a').on('hover', function () {
    var username = $(this).attr('href');

    showTooltipProfile(username, { below: $(this) });
  });
});
```

### What's wrong?

This anti-pattern leads to many issues, which rsjs attempts to address.

 * **Ambiguious sources:** It's not obvious where to look for the JS behavior. (Is the handler attached to *.author*, *.footnote*, or *.profile-link*?)

 * **Non-reusable:** It's not obvious how to reuse the behavior in another page. (Should *blogpost.js* be included in the other pages that need it? What if that file contains other behaviors you don't need for that page?)

 * **Lack of organization:** Making new behaviors gets confusing. (Do you make a new `.js` file for each page? Do you add them to the global `application.js`? How do you load them?)

### About Rails

This styleguide assumes Rails conventions of concatenating .js files. These don't apply to loaders like Browserify or RequireJS.

## Structure

### Think in component behaviors

Think that a piece of JavaScript code to will only affect 1 "component", that is, a section in the DOM.

There files are "behaviors": code to describe dynamic JS behavior to affect a block of static HTML. In this example, the JS component `collapsible-nav` only affects a certain DOM subtree, and is placed on its own file.

```html
<div class='main-navbar' role='collapsible-nav'>
  <button class='expand' role='expand'>Expand</button>

  <a href='/'>Home</a>
  <ul>...</ul>
</div>
```

```js
/* behaviors/collapsible-nav.js */
$(function () {
  var $nav = $('[role~="collapsible-nav"]');
  if (!$nav.length) return;

  $nav
    .on('click', '[role~="expand"]', function () {
      $nav.addClass('-expanded');
    })
    .on('mouseout', function () {
      $nav.removeClass('-expanded');
    });
});
```

### One component per file

Each file should a self-contained piece of code that only affects a *single* element type.

Keep them in your project's `behaviors/` path. Name these files according to the roles ([see below](#usetheroleattribute)) or class names they affect.

```
└── javascripts/
    └── behaviors/
        ├── collapsible-nav.js
        ├── avatar-hover.js
        ├── popup-dialog.js
        └── notification.js
```

### Load components in all pages

Your main .js file should be a concatenation of all your `behaviors`.

It should be safe to load all behaviors for all pages. Since your behaviors are localized to their respective components, they will not have any effect unless the element it applies to is on the page.

In Rails, this can be accomplished with `require_tree`.

```js
// js/application.js
/*= require_tree ./behaviors
```

### Use the role attribute

*Optional:* It's preferred to mark your component with a `role` attribute.

You can use ID's and classes, but this can be confusing since it isn't obvious which class names are for styles and which have JS behaviors bound to them. This applies to elements inside the component too, such as buttons (see collapsible-nav example).

```html
<!-- bad -->
<div class='user-info'>...</div>
$('.user-info').on('hover', function() { ... });
```

```html
<!-- ok -->
<div class='user-info' role='avatar-popup'>...</div>
$('[role~="avatar-popup"]').on('hover', function() { ... });
```

### Don't overload class names

If you don't like the `role` attribute and prefer classes, don't add styles to the classes that your JS uses. For instance, if you're styling a `.user-info` class, don't attach an event to it; instead, add another class name (eg, `.js-user-info`) to use in your JS.

This will also make it easier to restyle components as needed.

```html
<!--
Bad: is the JavaScript behavior attached to .user-info? or to
.centered? This can be confusing developers unfamiliar with your code.
-->
  <div class='user-info centered'>...</div>
  $('.user-info').on('hover', function() { ... });

<!--
Better: this makes it more obvious to find the source of
the behavior.
-->
  <div class='user-info js-avatar-popup'>...</div>
  $('.js-avatar-popup').on('hover', function() { ... });
```

## Writing code

These are conventions that can be handled by other libraries. For straight jQuery however, here are some guidelines on how to write behaviors.

### Consider using onmount

[onmount] is a library that allows you to write safe, idempotent, reusable and testable behaviors. It makes your JavaScript code compatible with Turbolinks, and it'd allow you to write better unit tests.

```js
$.onmount('.js-push-button', function () {
  $(this).on('click', function () {
    alert('working...')
  })
})
```

### Use document.ready

Add your events and initialization code under the [document.ready] handler.

```js
$(function() {
  $('[role~="expanding-menu"]').on('hover', function () {
    // ...
  });
});
```

The only time document.ready is not necessary is when attaching event delegations to `document`. These are typically best done _before_ DOM ready, which would allow them to work even when the entire document hasn't loaded completely.

```js
$(document).on('click', '[role~=".bar"]', function () {
  // ...
});
```

### Use each() when needed

When your behavior needs to either initialize first and/or keep a state, consider using [jQuery.each]. See [extras](extras.md#example-of-each) for a more detailed example.

If you are using [onmount], this is not necessary.

```js
$(function() {
  $('[role~="expanding-menu"]').each(function() {
    var $menu = $(this);
    var state = {};

    // - do some initialization code on $menu
    // - bind events to $menu
    // - use `state` to manage state
  })
})
```

### Avoid side effects

Make sure that each of your JavaScript files will not throw errors or have side effects when the element is not present on the page.

```js
$(function () {
  var $nav = $("[role~='hiding-nav']");

  // Don't bind the `scroll` event handler if the element isn't present.
  // This will avoid making the page sluggish unnecessarily. This also
  // avoids the error that $nav[0] will be undefined.
  if (!$nav.length) return;

  $('html, body').on('scroll', function () {
    if ($nav[0].disabled) return;
    var isScrolled = $(window).scrollTop() > $nav.height();
    $nav.toggleClass('-hidden', !isScrolled);
  });
});
```

### Dynamic content

There are times when you may wish to apply behaviors to content that may appear on the page later on. This is the case for AJAX-loaded content such as modal dialogs.

In this case, wrap your initialization code in a function, and run that function on document.ready and when the DOM changes (eg, `show.bs.modal` in the case of Bootstrap modal dialogs).

Since your initializer may be called multiple times in a page, you will need to make them [idempotent] by bypassing elements that it has already been applied to. This can be done with an [include guard] pattern.

An alternative to this would be event delegation: see [extras](extras.md#event-delegation) for more info.

[idempotent]: https://en.wikipedia.org/wiki/Idempotent
[include guard]: http://en.wikipedia.org/wiki/Include_guard

```js
;(function() {
function init() {
  $('[role~="key-value-pair"]').each(function () {
    var $parent = $(this);

    // an include guard to keep it idempotent
    if ($parent.data('key-value-pair-loaded')) return;
    $parent.data('key-value-pair-loaded', true);

    // init code here
  })
}

$(init);
$(document).on('show.bs.modal', init);
})();
```


## Namespacing

### Keep the global namespace clean

Place your publically-accessible classes and functions in an object like `App`.

```js
if (!window.App) window.App = {};

App.Editor = function() {
  // ...
};
```

### Organize your helpers

If there are functions that will be reusable across multiple behaviors, put them in a namespace. Place these files in `helpers/`.

```js
/* helpers/format_error.js */
if (!window.Helpers) window.Helpers = {};

Helpers.formatError = function (err) {
  return "" + err.project_id + " error: " + err.message;
};
```

## Third party libraries

If you want to integrate 3rd-party scripts into your app, consider them as component behaviors as well. For instance, you can integrate [select2.js] by affecting only `.js-select2` classes.

[select2.js]: https://github.com/select2/select2
[wow.js]: http://mynameismatthieu.com/WOW/

```js
...
└── javascripts/
    └── behaviors/
        ├── colorpicker.js
        ├── select2.js
        └── wow.js
```

```js
// select2.js -- affects `.js-select2`
$(function () {
  $(".js-select2").select2();
})
```

```js
// wow.js -- affects `.wow`
$(function () {
  new WOW().init();
})
```

### Separate your vendor libs

Keep your 3rd-party libraries in something like `vendor.js`.

This will have browsers cache the 3rd-party libs separately so users won't need to re-fetch them on your next deployment.

It also makes it easier to create new app packages should you need more than one (eg, one for your public pages and another for private dashboards).

```js
/* vendor.js */
//= require jquery
//= require jquery_ujs
//= require bootstrap
```

```js
/* application.js */
//= require_tree ./helpers
//= require_tree ./behaviors
```

### Load 3rd-party resources asynchronously

For third-party resources that are loaded externally, consider loading them with an asynchronous loading mechanism such as [script.js].

Some 3rd party libraries are loaded via `<script>` tags, such as Google Maps's API. These vendors typically expect you to embed them like so:

```html
<script src='//maps.google.com/maps/api/js?v=3.1.3&libraries=geometry'></script>
```

If your website doesn't use them everywhere, it'd be wasteful to include them on every page. Instead, consider using [script.js] wrapped in a helper function to load them on an as-needed basis.

```js
function useGoogleMaps (fn) {
  var url = '//maps.google.com/maps/api/js?v=3.1.3&libraries=geometry'

  $script(url, function () {
    // pass the global `gmaps` variable to the callback function.
    fn(window.gmaps)
  })
}
```

This helper can be used like so:

```js
useGoogleMaps(function (gmaps) {
  // use `gmaps` here
})
```

## Conclusion

This document is a result of my own trial-and-error across many projects, finding out what patterns are worth adhering to on the next project.

This document prioritizes developer sanity first. The approaches here may not have the most performant, especially given its preference for event delegations and running `document.ready` code for pages that don't need it. Regardless, the pros and cons were weighed and ultimately favored approaches that would be maintainable in the long run.

The guidelines outlined here are not a one-size-fits-all approach. For one, it's not suited for [single page applications][SPA], or any other website relying on very heavy client-side behavior. It's written for the typical conventional web app in mind: a collection of HTML pages that occasionally need a bit of JavaScript to make things work.

As with every other guideline document out there, try out and find out what works for you and what doesn't, and adapt accordingly.

## Further reading

- [rscss.io][rscss] - reasonable system for CSS stylesheet structure
- [onmount] - for safe, idempotent JavaScript behaviors and easy Turbolinks support
- [script.js] - for loading external scripts
- [kossnocorp/role](https://github.com/kossnocorp/role#use-role-attribute-ftw)'s explanation on using the `role` attribute

[document.ready]: http://api.jquery.com/ready/
[jQuery.each]: http://api.jquery.com/jQuery.each/
[extras]: extras.md
[onmount]: https://github.com/rstacruz/onmount
[script.js]: https://github.com/ded/script.js/
[rscss]: https://rscss.io
