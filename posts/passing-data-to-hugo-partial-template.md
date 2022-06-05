---
title: Passing data to a Hugo partial template
description: While working on setting up a photo gallery template for Hugo posts, I wanted to use structured data separated from content and pass this data to a template.
date: 2022-06-04
tags:
  - eleventy
layout: layouts/post.njk
templateEngineOverride: md
---

While working on setting up a photo gallery template for Hugo posts, I wanted to use structured data separated from content and pass this data to a template.

I found this post helpful - [Hugo: Insert data into content with a shortcode](https://input.sh/hugo-data-into-content-with-a-shortcode/). But my use case was slightly different. I was creating a new layout while trying to reuse an existing `partial` template in a Hugo theme.

## Photo Gallery Page

My photo gallery page contains `front matter` with information about the data and some simple content about the photos.

```md
---
title: "Hawaii Trip 2022"
date: 2022-06-04T19:52:37Z
draft: true
layout: gallery
photo_year: "2022"
photo_set: "hawaii-trip"
---

## Hawaii trip

We had a great vacation in Hawaii and here are some photos.
```

## Photo Data

I had the following data in `data/sets/2022/hawaii-trip.yml`. I wanted to keep the data structured as well as organized by year.

```yml
title: "Hawaii Trip 2022"
style: "style1 medium lightbox onscroll-fade-in"
location: "Oahu, HI, USA."
content: |
  This is a <strong>Photo Gallery</strong>.
pictures:
  - title: "Title"
    content: "Lorem ipsum dolor amet, consectetur magna etiam elit. Etiam sed ultrices."
    image: "/images/gallery/fulls/01.jpg"
    thumb: "/images/gallery/thumbs/01.jpg"
    button: "Details"

  - title: "Title"
    content: "Lorem ipsum dolor amet, consectetur magna etiam elit. Etiam sed ultrices."
    image: "/images/gallery/fulls/02.jpg"
    thumb: "/images/gallery/thumbs/02.jpg"
    button: "Details"

  - title: "Title"
    content: "Lorem ipsum dolor amet, consectetur magna etiam elit. Etiam sed ultrices."
    image: "/images/gallery/fulls/03.jpg"
    thumb: "/images/gallery/thumbs/03.jpg"
    button: "Details"

  - title: "Title"
    content: "Lorem ipsum dolor amet, consectetur magna etiam elit. Etiam sed ultrices."
    image: "/images/gallery/fulls/04.jpg"
    thumb: "/images/gallery/thumbs/04.jpg"
    button: "Details"
```

## Photo Gallery Layout Template

In my custom layout template, the data to read is based on `photo_year` and `photo_set` parameters. Hugo supports nested calls to the `index` function. This data is then passed as an argument to the 'gallery' `partial` template.

```go
{{ define "main" }}

{{ $gallery_data := index (index .Site.Data.sets .Params.photo_year) .Params.photo_set }}
{{ partial "gallery" $gallery_data }}
{{ .Content }}
{{ partial "template/footer" . }}

{{ end }}
```

## Theme Gallery Partial Template

Th 'gallery' `partial` is used as-is from [caressofsteel's hugo-story](https://github.com/caressofsteel/hugo-story/blob/master/layouts/partials/gallery.html) theme. Including the relevant code here for completeness.

```go
<!-- Gallery -->
<section class="wrapper style1 align-center">
    <div class="inner">
        <h2>{{ .title }}</h2>
        <p>{{ .content | safeHTML }}</p>
    </div>

    <!-- Gallery -->
    <div class="gallery {{ .style }}">
        {{ range .pictures }}
            {{ partial "picture" . }}
        {{ end }}
    </div>
</section>
```

## References

Some references I used for my setup

+ [Hugo: Insert data into content with a shortcode](https://input.sh/hugo-data-into-content-with-a-shortcode/)
+ [Mixing content and data in Hugo](https://michele.io/content-data-hugo/)
+ [Hugo and Data: The Basics](https://www.thenewdynamic.com/article/hugo-data/manipulation-and-logic-the-basics/)
+ [Front Matter](https://gohugo.io/content-management/front-matter/)
+ <https://stackoverflow.com/questions/71031296/hugo-change-layout>
+ [Hugo's Lookup Order](https://gohugo.io/templates/lookup-order/)
+ https://stackoverflow.com/questions/67482975/hugo-using-index-with-complex-key-to-get-data
+ [index function](https://gohugo.io/functions/index-function/)
+ <https://discourse.gohugo.io/t/string-variable-concatenation-to-build-path-for-partial/701>
