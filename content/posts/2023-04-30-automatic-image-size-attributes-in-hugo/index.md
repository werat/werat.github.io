---
title: "Automatic image size attributes in Hugo"
date:   2023-04-30 12:00:00 +0100
tags:
   - hugo
   - performace
   - web
---

It's [a good practice](https://web.dev/optimize-cls/#images-without-dimensions) to always include `width` and `height` attributes on images and video elements. Without them the browser has no way of knowing how large the image is going to be and cannot reserve the space in advance when rendering the page. This often leads to the content moving around during the page load, since images take extra time to fetch. This is called [Layout Shift](https://web.dev/cls/) and I find it extremely annoying!

This website is built with [Hugo](https://gohugo.io/) and one of the first things I noticed after running the [PageSpeed Insights](https://pagespeed.web.dev/) analyzer is that it actually doesn't set the size attributes on `<img>` tags! The website content is written in markdown (e.g. [this post](https://github.com/werat/werat.github.io/blob/956954c4c00055428167c3677d129c2a530e4848/content/posts/2023-04-30-automatic-image-size-attributes-in-hugo/index.md)), so in the source code it looks just like this:

```markdown
![AltText](image.png)
```

When building the website Hugo uses the markdown renderer to produce the corresponding HTML, which would look like this:

```html
<img src="image.png" alt="AltText">
```

As you can see, no `width` and `height` size attributes here. I'm not really sure why Hugo doesn't include them by default. Maybe it's because the referenced image might be malformed or even not exist, but in my blog I have control over the data and I prefer to have these attributes to be always present.

## Using markdown render hooks

The Markdown renderer used by Hugo supports [render hooks](https://gohugo.io/templates/render-hooks/), which allow the user to override markdown rendering functionality. In particular we can provide a custom template and replace the default boring image rendering logic with our own smart size-attributes-providing one.

The image render hook should be defined in `layouts/_default/_markup/render-image.html`. You can have different hooks for different sections and output formats, but I don't really care about that for my simple blog. Extracting the image dimentions is pretty simple, so the whole template is quite short:

```html
{{- $image := .Page.Resources.GetMatch (printf "%s" (.Destination | safeURL)) -}}
<img
    src="{{ .Destination | safeURL }}" alt="{{ .Text }}"
    {{ with $image }}
    width="{{ $image.Width }}" height="{{ $image.Height }}"
    {{ end }} />
```

This template was perfect for me for a while, but at some point I wanted to set the `max-width` property, because some images were too big. There's no good way to pass the arbitrary parameters to the render template, but luckily the markdown syntax for images allows providing an image "title":

```markdown
![AltText](image.png "Title")
```

I don't set titles for my images, so instead I repurposed this parameter for setting the maximum width:

```html
{{- $image := .Page.Resources.GetMatch (printf "%s" (.Destination | safeURL)) -}}
<img
    src="{{ .Destination | safeURL }}" alt="{{ .Text }}"
    {{ with $image }}
    width="{{ $image.Width }}" height="{{ $image.Height }}"
    {{ end }}
    {{ with .Title}} style="width: 100%; max-width: {{ . }};"{{ end }}
    />
```

With this template the size attributes are provided automatically and `max-width` is set conditionally, if the "title" is provided. So now `![AltText](image.png "500px")` is rendered like this:

```html
<img
    src="image.png" alt="AltText"
    width="667" height="375"
    style="width: 100%; max-width: 570px;"/>
```

Here's the final template used for this website -- [layouts/_default/_markup/render-image.html](https://github.com/werat/werat.github.io/blob/faa5bab6f6498b25ce0df1c261e16af7fa881f9d/layouts/_default/_markup/render-image.html).
