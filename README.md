This article will discuss responsive image markup and twig macros to automate generating that markup.

There are three things I hope you will get from this article:

1. A quick overview of responsive images (and links to further resources).
2. A useful twig macro for generating responsive image markup.
3. Nerdy details about programming in twig, useful when you write your own twig macros.

## Responsive Images

Here are some excellent articles with information on responsive images. If you know nothing about responsive images, read at least one of them, and then come back:
- [Figuring Out Responsive Images](https://css-tricks.com/video-screencasts/133-figuring-responsive-images/), _CSS-Tricks_
-  [Responsive Images Done Right: A Guide To &lt;picture&gt; And srcset](http://www.smashingmagazine.com/2014/05/14/responsive-images-done-right-guide-picture-srcset/) _Smashing Magazine_
-  [Srcset and sizes](https://ericportis.com/posts/2014/srcset-sizes/), _Eric Portis_
- [Responsive Images in Practice](http://alistapart.com/article/responsive-images-in-practice), _A List Apart_
- [Picturefill polyfill](http://scottjehl.github.io/picturefill/), _Picturefill_

There are a number of responsive image use cases. This article will focus on two, both of which use the `img` tag (and not the `picture` tag).

### Fixed-width images

The first use case is a fixed-width image (so not actually responsive...) that adapts to different device-pixel-ratios. Say you have a thumbnail image which is always displayed 200px wide, and you have versions for 1x (thumb.jpg) and 2x (thumb_2x.jpg). The markup for this is:

``` html
<img
  src="thumb.jpg"
  srcset="thumb_2x.jpg 2x"
  alt="thumb"
/>
```

Browsers that don't understand `srcset` will ignore that attribute, and download `thumb.jpg`. Browsers that do understand `srcset` will use `thumb.jpg` on devices with `1x` resolution, and `thumb_2x.jpg` on devices with `2x` resolution (or higher).

### Variable-width images

The second use case is variable-width images with no art direction (so the only difference in the images is the resolution). Say you have an image that will be full-width on narrower screens (&lt;= 30em), and 25em wide on larger screens. And you have this image available in three different sizes:

- illo_small.jpg is 400px
- illo_medium.jpg is 800px
- illo_large.jpg is 1000px

The markup for this is:

``` html
<img
  src="illo_medium.jpg"
  srcset="illo_small.jpg 400w, illo_medium.jpg 800w, illo_large.jpg 1000w"
  sizes="(maxwidth 30em) 100vw, 25em"
  alt="illustration"
/>
```

Browsers that don't understand `srcset` and `sizes` will use the 800px version. Browsers that do understand `srcset` and `sizes` will know from the `srcset` attribute that the image is available in those three sizes, and from the `sizes` attribute that if the browser window is up to 30em the image will be sized at `100vw` (which is 100% of the viewport width), and otherwise it will be `25em`. At this point it is up to the browser to decide which one to download.

## A very brief review of twig macros

Here is a simple [twig macro](http://twig.sensiolabs.org/doc/tags/macro.html) definition:

``` twig
{% macro greet(name='World') %}
  Hello, {{name}}!
{% endmacro %}
```
Every macro begins with the `{% macro %}` tag, and ends with the `{% endmacro %}` tag. This one takes one parameter (`name`), which has a default value of `'World'`.
To call it, you first need to [`import`](http://twig.sensiolabs.org/doc/tags/import.html) it.

``` twig
{% macro greet(name='World') %}
  Hello, {{name}}!
{% endmacro %}

{# Import the macro into the file in which it is defined #}
{% import _self as self %}

{# Call your macro #}
{{ self.greet() }}
```
The example above will generate `Hello, World!`. To generate a different output, you can call the macro using a different `name` parameter.

``` twig
{# This will generate `Hello, Dear Reader!` #}
{{ self.greet('Dear Reader') }}
```

If the macro is defined in a different file than the one where it is called, you import it (in the calling file):

``` twig
{% import '_macros/_utils' as m_utils %}
{# Call an imported macro using the name you imported it with #}
{{ m_utils.greet() }}
```

What you name your macro files, and what you import them as are matters of convention. I use `self` when the macro is in the same file, and `m_whatever` when the macro is in the file `_macros/_whatever.twig`.

## Twig macros for responsive images

A lot of the work in creating the responsive image markup can be automated. Rather than uploading the same image at different resolutions, we can use Craft's [Image Transformations](http://buildwithcraft.com/docs/image-transforms) to generate the various image sizes.

I have a macro file, `_img.twig` that has macros for each of the two use cases ([available on github](https://github.com/marionnewlevant/twig-img-macro/blob/master/_img.twig)). You pass these macros an image asset, and they generate markup for a responsive image.

One thing these macros do is try to be smart about which image transforms to ask Craft for. There is no point in Craft transforming an image larger than the original size. If you have a 400px image, and you need an 800px image, the browser should make that transformation. There is also no point in Craft transforming an image to the same size. If the image is originally 800px, do not transform it to 800px, just use the original image.

`_img.twig` defines two public macros (and a third for use only inside the file):
- **fixedSize()** - Public
- **responsive()** - Public
- **_classAttr()** - Internal

### Example of fixed-width image macro

`fixedSize()` is for the first use case - a fixed size image that adapts to different device-pixel-ratios. Here is an example of using it:

``` twig
{% import '_macros/_img' as m_img %}
{% set thumbAsset = entry.assetField.first() %}

{# Example of a simple call to the macro #}
{{ m_img.fixedSize(thumbAsset, 200) }}

{# Example of more advanced call
   specifying alt attribute of 'some thumb' and class of 'thumb-class'
#}
{{ m_img.fixedSize(thumbAsset, 200, {alt: 'some thumb', class: ['thumb-class']}) }}
```

### Example of responsive-width image macro

`responsive()` is for the second use case - a variable width image. Here is an example of using it:

``` twig
{% import '_macros/_img' as m_img %}

{% set illoAsset = entry.assetField.first() %}

{# Example of a simple call to the macro #}
{{ m_img.responsive(illoAsset) }}

{# Example of another macro call using a different style #}
{{ m_img.responsive(thumbAsset, {style: 'thumb'}) }}

```
This is not quite all there is to using the `responsive()` macro. You also need to define what widths the image should be avaiable in (`srcset`), and how wide the image will be displayed (`sizes`). Details of that are below.

## Taking a closer look at the _img.twig macro file

Here is an annotated version of the twig macro file, `_img.twig`. The annotations follow the code they describe.

### The Class Attribute Macro: __classAttr()_

``` twig
{% macro _classAttr(classes) %}
  {%- if (classes|length) -%}
    class="{{ classes|join(' ') }}"
  {%- endif -%}
{% endmacro %}
```

`_classAttr` is a macro intended to only be called from within `_img.twig`, which is why its name begins with `_` (this is a convention of mine, not something the language enforces). If it turns out to have wider utility, I will factor it out into a file of utility macros (and rename it to `classAttr`).

`_classAttr` takes an array of class names, and returns a class attribute string. `_classAttr(['foo', 'bar'])` will return `class="foo bar"`. `_classAttr([])` will return an empty string. `{%-` and `-%}` are twig's [tag level whitespace control](http://twig.sensiolabs.org/doc/templates.html#whitespace-control), used to strip out whitespace the macro would otherwise generate. The [join](http://twig.sensiolabs.org/doc/filters/join.html) filter takes an array of strings and turns it into a string of space separated words.

### The Fixed Size Macro: _fixedSize()_

``` twig
{% macro fixedSize(asset, width, options={}) %}
{% import _self as self %}
```

This is the macro for the first use case. It is passed an [asset](http://buildwithcraft.com/docs/assets), the width in `px` at which the image will be displayed, and [optionally] a hash of options.

Since we are going to use the `_classAttr` macro, defined in this file, we [`import`](http://twig.sensiolabs.org/doc/tags/import.html) it.

``` twig
{% set options = {
  alt: asset.altText,
  class: []
}| merge(options) %}
```

We define default options, and [`merge`](http://twig.sensiolabs.org/doc/filters/merge.html) them with the `options` which were passed in. Options passed in will override these default ones.

The two options are `alt`, and `class`. `alt` will be the alt text for the tag. All my image assets have a field `altText`, which is used as the default value. `class` is an array of class names, defaulting to none.

``` twig
{% set transform = {
  mode: 'stretch'
} %}
```

We define the [transform](http://buildwithcraft.com/docs/image-transforms#defining-transforms-in-your-templates) we will use. This `transform` is missing the `width` attribute. When we actually use it, we will `merge` in the `width` attribute. So to transform to 200: `asset.getUrl(transform|merge({width: 200}))`. (The alternative would be `asset.getUrl({mode: 'stretch', width: 200})`, but I think using the `transform` variable is slightly easier to read).

``` twig
{% set nativeWidth = asset.getWidth(false) %}
```

Set `nativeWidth` to the original, untransformed width of the image ([`getWidth(false)`](http://buildwithcraft.com/docs/templating/assetfilemodel#getWidth) returns the original width).

``` twig
<img
  {# src attr: transformed only if necessary #}
  {% if nativeWidth <= width %}
    src="{{ asset.getUrl(false) }}"
  {% else %}
    src="{{ asset.getUrl(transform|merge({width: width})) }}"
  {% endif %}

  {# srcset attr, but only if 2x <= nativeWidth #}
  {% if width*2 == nativeWidth %}
    srcset="{{ asset.getUrl(false) }} 2x"
  {% elseif width*2 < nativeWidth %}
    srcset="{{ asset.getUrl(transform|merge({width: width*2})) }} 2x"
  {% endif %}

  alt="{{options.alt}}"
  {{ self._classAttr(options.class) }}
/>
{% endmacro %}
```

Here we generate the `img` tag. First is the `src` attribute, which will be transformed down to the 1x size only if the `nativeWidth` is larger than that. Second is the `srcset` attribute. There are three cases here:

- the `nativeWidth` exactly matches the 2x size - no transform
- `nativeWidth` is larger than the 2x size - transform it down
- `nativeWidth` is smaller than the 2x size - no `srcset` at all

Finally, we have our `alt` and `class` attributes.


### The Responsive Macro: _responsive()_

``` twig
{% macro responsive(asset, options={}) %}

{% import _self as self %}

  {% set options = {
    alt: asset.altText,
    class: [],
    style: 'default'
  }| merge(options) %}

  {% set transform = {
    mode: 'stretch'
  } %}

  {% set nativeWidth = asset.getWidth(false) %}
```

This is the macro for the second use case. It is passed an asset and an options hash. In addition to the `alt` and `class` values, there is a `style` (which defaults to `default`).

The `style` is used to pick the configuration for a particular style of responsiveness from the `config` hash (defined below).

We set `options`, `transform`, and `nativeWidth` just as in `fixedSize()`.

``` twig
{#
 # Here is where you configure the image styles.
 # You are going to have to modify this for your
 # individual site.
 #
 # config is a hash, where the key is the style,
 # and the value is another hash
 # of sizes, srcsetWidths, and defaultWidth.
 #
 # There should always be a 'default' style.
 # Redefine the 'default' to whatever makes sense
 # for you, and add other styles as needed.
 #
 # srcsetWidths: image widths that should appear
 #   in the srcset.
 # sizes: media queries that specify the widths
 #   of the image at different screen widths.
 #   The first one that matches is used.
 # defaultWidth: image width for the src image
 #   (fallback for browsers that don't understand
 #   srcset)
 #}

{% set config = {
  default: {
    srcsetWidths: [400, 800, 1000],
    sizes: [
      '(max-width: 30rem) 100vw',
      '25em'
    ],
    defaultWidth: 800
  },
  thumb: {
    srcsetWidths: [200, 400],
    sizes: [
      '200px'
    ],
    defaultWidth: 200
  }
} %}
```

`config` will need to be modified for your particular project. Here is where you specify the widths for `srcset` and the values for `sizes`. This `default` style is the same as the one from the `illo` example earlier in the article. The `thumb` style is another way of doing the 200px fixed width thumbs.

``` twig
{% set params = config[options.style] %}
```

`config` defines multiple sets of style params. Fetch the one we will use.

``` twig
{% set srcset = [asset.getUrl(false)~' '~nativeWidth~'w'] %}
```

`srcset` will be an array of strings (we will use `join` to turn them into one comma separated string later). Here we initialize the array with a single string for the `nativeWidth` image, which is always included. `~` is twig's [concatenation operator](http://twig.sensiolabs.org/doc/templates.html#other-operators). We concatenate the image url, a space, the width, and the string `w`, and set `srcset` to an array of just that string.

``` twig
{% for width in params['srcsetWidths'] %}
  {% if width < nativeWidth %}
    {% set srcset
       = srcset|merge([asset.getUrl(transform|merge({width: width}))~' '~width~'w'])
    %}
  {% endif %}
{% endfor %}
```

Here we loop over the `srcsetWidths`, and add the width string to the `srcset` array for any that are less than the `nativeWidth`.

``` twig
<img
  {# src attr: transformed only if necessary #}
  {% if nativeWidth <= params['defaultWidth'] %}
    src="{{ asset.getUrl(false) }}"
  {% else %}
    src="{{ asset.getUrl(transform|merge({width: params['defaultWidth']})) }}"
  {% endif %}

  srcset="{{ srcset|join(', ') }}"
  sizes="{{ params['sizes']|join(', ') }}"

  alt="{{options.alt}}"
  {{ self._classAttr(options.class) }}
/>
{% endmacro %}
```

Here we generate the `img` tag. First is the `src` attribute, which will be our `defaultWidth` if the image is large enough, and otherwise the `nativeWidth`. After that is the `srcset` attribute, where we `join` the `srcset` variable, this time putting `", "` between the strings. Then the `sizes` attribute, similarly constructed with `join`. And finally the `alt` and `class` attributes.

## Some Details

* Responsive images are reasonably well supported by browsers (see [caniuse data](http://caniuse.com/#search=srcset) for details). Additionally, there is a [polyfill](http://scottjehl.github.io/picturefill/) which is very easy to use. Download it, and add this script tag:
``` html
<script src="{{siteUrl}}js/vendor/picturefill.min.js" async></script>
```

* I always set [generateTransformsBeforePageLoad](http://buildwithcraft.com/docs/config-settings#generateTransformsBeforePageLoad) to `true` in my `config/general.php`:
``` php
'generateTransformsBeforePageLoad' => true,
```
This generates the transform when `getUrl()` is called, rather than waiting for the browser to request the image. This will make the first page load a little bit slower, but it lets you [`cache`](http://buildwithcraft.com/docs/templating/cache) the result of calling the macros.

* You will need to specify the `config` styles for the `responsive()` macro, but you can wait to fill in the details of the styles until your site design is quite solid. And these values never have to be completely precise. Reasonably close is fine (though you probably want to get the breakpoint values exact).

* Enjoy! Feel free to contact me with any questions: [marion.newlevant@gmail.com](mailto:marion.newlevant@gmail.com).
