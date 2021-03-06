# django-flexible-images, a responsive image solution for Django

## Quick usage

1. Add `flexible_images` to your `INSTALLED_APPS`.
2. Load the `flexible_images` template tag library.
3. Load `flexible-images/flexible-images.js` in your template.
4. Use `{% flexible_image your_model_instance.some_imagefield %}` inside a
   container of the desired width.

For sample usage and for testing, this comes with a `FlexibleImageTestView`
view class in `views.py` and sample 3:2 (ish) images as
`static/responsive-test-image-1.jpg` and
`static/responsive-test-image-2.jpg`. (Copy these to your MEDIA_ROOT;
ImageFile gets upset if you try to open a file outside of it.)


## What does it do?

### `srcset`, for browsers that fully support it.

Right now, current Chrome, Edge, Firefox and iOS Safari have full support for
the `srcset` attribute. This allows a page to specify a list of image
sizes that it has available and for the client to pick the most appropriate
one.

### Deferred JavaScript image switching, for browsers that don't.

Some browsers don't support `srcset`. Others do, but don't support it well
(they only support pixel density queries). Both cases should be reliably
detected.

For browsers that do not fully support `srcset`, the lowest-resolution image
will be displayed first. When the document is loaded, JavaScript will detect
if the user's screen (actually, the width of the parent element of the image)
merits switching images to a higher resolution version.

It will also do this when a window is resized (including device rotation). It
will never replace a higher-resolution image with a lower-resolution one.

This was inspired by
[django-responsive-images](https://github.com/onespacemedia/django-responsive-images),
but it is implemented somewhat differently.


### Smarter aspect ratio preservation

I consider this the most important feature, in that it drastically reduces
the time taken to get to a ready state on the document.

We want to ensure that images will fit on any screen size without overflowing,
but preserve their aspect ratio.

The simple approach is to use `max-width: 100%` and `height: auto` on an
image. However, because the browser *cannot know in advance how high
`auto` will be*, when an image starts to load it causes a reflow of the
document. This is usually experienced as document elements being pushed down
as images load. Most commonly, this means that the thing you are reading
will be pushed down as images above it load.

Fortunately, two quirks of the CSS specification are here to help us.

1. When specified as a percentage, vertical `padding` of CSS elements is
   calculated as a percentage of the parent element's width.
2. When specified as a percentage, the height of an absolutely positioned
   element within a relative-positioned element will be the height of the
   bounding box of the image, which is to say, the inner height of the box
   *plus its padding*.

Therefore, 1) will allow us to create a container with a certain aspect ratio
(e.g. if the box has `padding-bottom` set to 66.666%, the container will have
a 3:2 aspect ratio). This will be the case at any size, and this size is
computed immediately as the element is added into the document; no reflows
will occur.

From there, if we absolutely position an image to the top left of that and
set both width and height to 100%, it will fill that container, and it will
preserve its aspect ratio at any screen size.

The code for this is trivial; if you're only interested in this part I would
suggest ripping it out for use in your own projects.

If you want to take care of padding and sizing yourself, pass
`no_css=True` to the template tag.

## How?

For most uses, just pass any ImageField as an argument to the
`{% flexible_image %}` template tag.

There are a few arguments you can pass to the tag if required:

* `container`: The HTML tag in which to wrap the image. Defaults to `div`.
* `no_css`: Disables adding inline CSS. Use this if you're taking care of
  sizing the image yourself. Defaults to `False`. Note that if you're doing
  this and you do not have `sorl-thumbnail` installed, `{% flexible_image %}`
  is effectively a no-op!
* `classes`: A space-separated list of extra classes to apply to the container
  element. Defaults to empty string.
* `inner_classes`: A space-separated list of exta classes to apply to the
  image itself (which is wrapped in the container to which `classes` is
  applied). Defaults to empty string.
* `alt`: [Alt text](https://en.wikipedia.org/wiki/Alt_attribute) for the
  image. For purely decorative images, this should be an empty string (the
  default). For images that convey information that can be textually described,
  this should be present.
* `background_image`: Set this to True to set the image as a CSS
  `background-image` on an empty &lt;div&gt;. Note that if this is set,
  JavaScript image switching will be in effect as there is not an equivalent
  of `srcset` for CSS background images.
* 'aria_hidden': Set to True if you wish to set the `aria-hidden` attribute
  on the container. Use this for purely decorative images. Defaults to
  `False`.

Don't use the value of `container` or `classes` as selectors for styling the
width of the image's container element with CSS; while some browsers
(incorrectly) calculate padding-bottom percentages from the width of the
element, others (correctly) calculate it from the width of the parent element.
Wrap your `{% flexible_image %}` tag in a container of the desired width
instead.

## The less opinionated version

If you don't like the default HTML implementation, or it does not work for
your situation, try using the `{% flexible_image_list %}` assignment tag. It
takes an ImageField as its only argument and returns a dictionary of this form
this from which you can build your own implementation:

```
[
  {
    'url': u'/media/cache/7b/6d/7b6dc5274f17a909387b94371f033953.jpg',
    'width': 480,
    'height': 318
  },
  {
    'url': u'/media/cache/97/8d/978d13a6ba734e970e42fe9e7415ca40.jpg',
    'width': 768,
    'height': 509,
  },
  [...]
]
```


## Settings

flexible-images has two settings for your settings.py.

**FLEXIBLE_IMAGE_ENGINE** can be set to `'sorl'` or to `None`. It defaults to
`'sorl'` if you have not set it. `None` means that you will only get the
aspect ratio padding mentioned above, not `srcset` and its JS fallback.

**FLEXIBLE_IMAGE_SIZES** defines the sizes that flexible-images will generate,
as a list of image widths in pixels, in ascending size order. This is
optional, and the defaults are as follows:

```
FLEXIBLE_IMAGE_SIZES = [
  480,
  768,
  1024,
  1280,
  1440,
]
```


## Who?

This was written by Lewis Collard at
[Onespacemedia](http://www.onespacemedia.com/).

[Daniel Samuels](http://danielsamuels.co.uk/) provided valuable guidance to get this to 1.0.


## Compatibility and requirements

You will need
[sorl-thumbnail](https://sorl-thumbnail.readthedocs.org) for `srcset`
(and the JavaScript switcher fallback) to work.

It is CSS-framework-agnostic; it'll work with any framework, or no framework.

The JavaScript is [vanilla JS](http://vanilla-js.com/), so it neither requires
a JavaScript framework, nor cares about the one you are using.

This should work with any recent version of Django. This has been tested with
1.8, but any earlier version should work fine.

## Browser support
This has been tested on several browsers:

* Internet Explorer 9+
* Chrome (any semi-recent version)
* Firefox (any semi-recent version)
* Safari 8 (OS X)
* iOS 8.1

Android should work fine as well (minimally tested), as will any semi-recent
WebKit-based browser.


## To-do

* easy_thumbnails support.
* jinja2 support.

## License

[Public domain](https://creativecommons.org/publicdomain/zero/1.0/).
