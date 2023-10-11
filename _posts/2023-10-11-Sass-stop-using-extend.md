---
layout: post
title: "Sass: stop using @extend"
---

*the following is an article I originally wrote for internal use at [Odoo](https://odoo.com), it has been minimally edited to make it understandable by people outside the organization.*

The `@extend` directive in Scss is a footgun, it's easy to misuse and difficult to use well, this article will go into why you're probably using it wrong.

## What @extend does

`@extend` takes a selector or placeholder, and makes it so that the selector of the current scope becomes equivalent to the extended selector in all contexts.

## What you probably think this means

You may think that if you want something to look like a bootstrap button, you can just extend the `btn` class, i.e.

```html
<div class="my-btn"></div>
```

```scss
.my-btn {
  @extend .btn;
}
```

You probably think that this will somehow copy all the styles of the bootstrap `btn` class in the current context.

## What it actually means

Going back to what I wrote earlier

> the selector of the current scope becomes equivalent to the extended selector in ***all contexts***

What this means is that any time the `btn` class is used in the scss of the current compilation unit, the Sass compiler will copy the selector that contains it, and replace the `btn` class with the current scope selector. This causes a few issues.

### Slow compilations and bloated output

When first encountering your `@extend` directive, the Sass compiler has to scan all the already generated rules so that it can copy-paste those containing `.btn` and replace it with your selector instead. For the rest of the compilation, any time any scss file in the compilation unit uses the `btn` class, it needs to duplicate the selector, and in some cases, the selector has to be duplicated more than once, e.g. when it contains a direct child or next sibling combinator (`>` or `+`). This means that you're making styling the class you're extending more expensive everywhere else for everyone.

Not only does this cause compilation itself to be extremely slow, it also generates loads of completely useless css:

#### Without @extend
<table>
<tr>
<td> SCSS </td> <td> Generated CSS </td>
</tr>
<tr>
<td>
{% highlight scss %}
// in bootstrap
.btn {
  some-rule: some-value;
}
// in existing code
.o_form_view {
  .o_row {
    > .btn {
      some-rule: some-value;
    }
  }
}
// in your code
.some_class {
  .some_other_class {
    div.my-btn {
      some-rule: some-value;
    }
  }
}
{% endhighlight %}
</td>
<td>
{% highlight css %}
.btn {
  some-rule: some-value;
}

.o_form_view .o_row > .btn {
  some-rule: some-value;
}

.some_class .some_other_class div.my-btn {
  some-rule: some-value;
}






/**
 * Weight: 161 bytes
 */
{% endhighlight %}
</td>
</tr>
</table>

#### With @extend

<table>
<tr>
<td> SCSS </td> <td> Generated CSS </td>
</tr>
<tr>
<td>
{% highlight scss %}
// in bootstrap
.btn {
  some-rule: some-value;
}
// in existing code
.o_form_view {
  .o_row {
    > .btn {
      some-rule: some-value;
    }
  }
}
// in your code
.some_class {
  .some_other_class {
    div.my-btn {
      @extend .btn;
    }
  }
}
{% endhighlight %}
</td>
<td>
{% highlight css %}
.btn,
.some_class .some_other_class div.my-btn {
  some-rule: some-value;
}

.o_form_view .o_row > .btn,
.o_form_view .some_class .some_other_class
.o_row > div.my-btn,
.some_class .some_other_class .o_form_view
.o_row > div.my-btn {
  some-rule: some-value;
}



/**
 * Weight: 260 bytes
 * (and it gets worse the more selectors
 * you already have targetting .btn)
 */
{% endhighlight %}
</td>
</tr>
</table>

Notice how the generated code now contains these selectors:
```css
.o_form_view .some_class .some_other_class .o_row > div.my-btn,
.some_class .some_other_class .o_form_view .o_row > div.my-btn
```
Even though the `my-btn` class will probably never be used in that context.

Considering that every time you extend the `btn` class, your selector will be copy-pasted in all the places that target the `.btn` selector, if you have 250 such selectors (as is the case in Odoo at the time of writing) this means that for every four characters in your selector, you're adding 1kb of compiled css. For a nested selector whose current selector scope is 40 characters long, which is the length of the selector in the example above, that's 10kb of generated css for a single `@extend` directive.

Not only is this bad for load times as there is more css to transmit over the wire, it also forces the browser to parse all that css, and means the browser has to check elements against those classes which slows down page layout.

### Breaking unrelated code

In addition to some combinators generating even more code, with some pseudo-selectors like `:not`, extending a class will add extra pseudo-selectors to the existing selectors which will **increase the specificity** of existing selectors and can break *existing styles*:

<table>
<tr>
<td> SCSS </td> <td> Generated CSS </td>
</tr>
<tr>
<td>

{% highlight scss %}
// Somewhere in existing code
a:not(.btn) {
  some-rule: some-value;
}

a:not(.other-class) {
  some-rule: some-overriding-value
}

// In code you're introducing
.my-btn {
  @extend .btn;
}
 {% endhighlight %}

</td>
<td>
    
{% highlight css %}
// Without your code
a:not(.btn) {
  some-rule: some-value;
}
a:not(.other-class) {
  some-rule: some-overriding-value;
}
// With your code
a:not(.btn):not(.my-btn) {
  some-rule: some-value;
}
a:not(.other-class) {
  some-rule: some-overriding-value;
}
{% endhighlight %}

</td>
</tr>
</table>

Without `@extend`, both selectors have the same specificity, and the second rule is be applied, because it appears later in the style sheet. By adding that `@extend`, the first rule is now *more specific*: you've changed how the existing style behaves, and potentially broke styles elsewhere.

## What to do instead

If you're trying to extend a bootstrap utility class, you probably should just use that utility class on your element instead, and only have the extra rules or overrides you want on your custom class:

```html
​<div class="btn my-btn"/>
```
```scss
​.my-btn {
  ​some-rule: some-value;
}
```

If that's really not an option, you can just write the few css rules you actually need. You don't update bootstrap every day, your rules will not drift from the bootstrap ones unless you upgrade bootstrap and when you do, you will have many problems to fix anyway. In a lot of cases, if you look at the bootrap code for the selector you're extending, they use a mixin or scss variables, so you can just reuse that, and if/when you update bootstrap, you will get the benefit of the updated mixin/variables anyway.

```scss
// instead of this
.my-btn {
  @extend .btn-sm;
}
// do this
.my-btn {
  @include button-size(
    $btn-padding-y-sm,
    $btn-padding-x-sm,
    $btn-font-size-sm,
    $btn-border-radius-sm
  );
}
```

## Exceptions

If you're trying to extend one of your own classes to share some of the rules between multiple selectors, you should instead declare a Sass placeholder and extend that. Extending placeholders doesn't suffer from the same problems, because placeholders will not be used in other places to style elements, and they don't emit any css of their own.

```scss
// instead of this
.my-btn-1 {
  lots: of;
  rules: you;
  want: to;
  share: between;
  multiple: classes;
}
.my-btn-2 {
  @extend .my-btn-1;
}
// do this
%my-btn {
  lots: of;
  rules: you;
  want: to;
  share: between;
  multiple: classes;
}
.my-btn-1 {
  @extend %my-btn;
}
.my-btn-2 {
  @extend %my-btn;
}
```

The other niche case where using `@extend` is useful is when renaming a class, but for some *good* reason, wanting to continue supporting the old class and apply the new style to it. In that case, make the old class extend the new class and add a deletion policy for this legacy compatibility class in a comment (ie: "this class exists for backwards compatibility and should be deleted in major version XX") so that there is a clear criterion for removal.

