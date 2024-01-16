# How to build a blog using Directus and Hugo

## Outline

## Introduction

This tutorial shows you how to build a blog website using Hugo and Directus. Hugo is a popular Static Site Generator (SSG) and Directus is a popular Headless Content Management System (CMS).

You will learn how to use Directus as a backend to create, edit and store articles for your blog. Hugo will be used to retrieve and render the articles as a frontend for your blog.

## Before You Start

- Install [Go](), [Hugo]() and a code editor for your computer.
- Sign up for a Directus Cloud account or self-host Directus.
- Some knowledge of Go is helpful but you can still follow along even if you haven't used Go before.

Create a new Directus Cloud project - any tier and configuration is suitable for this tutorial. 

Alternatively, launch a self-hosted Directus project on your computer. [Follow this guide]().

Open your terminal and run the following command to create a new Hugo website:

```bash
hugo new site my-website
```

Open `my-website` in your code editor.

## Using Global Metadata and Settings

In your Directus project, navigate to Settings > Data Model and create a new collection called `global`. Select 'Treat as a single object' under the Singleto option.

Create two text input fields - one with the key of `title` and the other with `description`.

Go to the Content module and select the Global collection. Enter information for the title and the description and hit Save.

![Hugo global collection configuration](/hugo-global-config.png)

By default, new collections are not accessible to the public. Navigate to Settings -> Access Control -> Public and give Read access to the Global collection.

### Configure Hugo Home Page

Inside `my-website` open the `config.toml` file and append the URL to your Directus project.

```toml
[params]
    api_url = "https://directus.example.com"
``` 

Create a new file in the `layouts` directory named `index.html`.

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">
    {{ $global := getJSON (printf "%s/items/global" site.Params.api_url) }}
    <title>{{ $global.data.title }}</title>
</head>

<body>
    <h1>{{ $global.data.title }}</h1>
    <p>{{ $global.data.description }}</p>
</body>

</html>
```

Type `hugo serve` in your terminal and Hugo will build your site. Open your site at http://localhost:1313 in your browser. You should see data from your Directus Global collection in your page. 

## Creating Pages With Directus

Create a new collection called `pages` - make the Primary ID Field a "Manually Entered String" called `slug`, which will correlate with the URL for the page. For example `about` will later correlate to the page localhost:1313/about.

Create a text input field called `title` and a WYSIWYG input field called `content`. In Access Control, give the Public role read access to the new collection. Create 3 items in the new collection - [here's some sample data](https://github.com/Marktawa/examples/tree/main/hugo-directus/demo-data/pages).

Inside `my-website` create a new directory named `pre-build`. This `pre-build` directory will be used to store the data fetched from the Directus backend. Add a `prebuild/config.toml` file. 

Open up `prebuild/config.toml` in your code editor and add the following:

```toml
disableKinds = ["sitemap", "taxonomy", "term"]

[outputs.home]
  to = ["html"]

[params]
  api_url = "https://directus.example.com"
```

This ensures the prebuild directory outputs only the data fetched from the Directus project API.

Inside of `pre-build`, create a new directory named `layouts` and add a file called `index.html`. This template file will be in charge of fetching the API data from the Directus project.

```html
{{ with resources.GetRemote (printf "%s/items/pages" site.Params.api_url) }}
    {{ $pages := unmarshal .Content }}
    {{ range $pages.data }}
        {{ $meta := dict "slug" .slug "title" .title }} 
        {{ $metaJson := jsonify $meta }}
        {{ $content := .content | safeHTML }}
        {{ $output := printf "%s\n%s" $metaJson $content }}
        {{ $filename := printf "pages/%s.md" .slug }}
        {{ $resource := resources.FromString $filename $output }} 
        {{ $file := $resource.RelPermalink }} 
    {{ end }}
{{ end }}
```

Run `hugo` from the `prebuild` directory. This will generate a directory named `prebuild/public/pages` containing all the pages from the Directus collection.

Mount the `prebuild/public/pages` directory for Hugo to read the pages collection. Open up your Hugo site's config file `config.toml` and append the following code:

```toml
[markup.goldmark.renderer]
  unsafe = true

[module]
  [[module.mounts]]
    source = "content"
    target = "content"

  [[module.mounts]]
    source = "prebuild/public/pages"
    target = "content"
```

Open the `layouts` directory and create a directory named `_default`. Inside `_default` add a file named `single.html`.

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">
    <title>{{ .Title }}</title>
</head>

<body>
    <h1>{{ .Title }}</h1>
    {{ .Content }}
</body>

</html>
```

This code will make sure your templates for the pages are using the proper Front Matter keys.

Run `hugo serve` from the root of your main Hugo project directory. Visit http://localhost:1313/about, replacing `about` with any of your item slugs. The page should show the data in your Directus pages collection.

## Creating Blog Posts With Directus

Create a new collection called `authors` with a single text input field called `name`. Create one or more authors.

Then, create a new collection called `posts` - make the Primary ID Field a "Manually Entered String" called `slug`, which will correlate with the URL for the page. For example `hello-world` will later correlate to the page localhost:1313/blog/hello-world.

Create the following fields in your posts data model:

- a text input field called `title`
- a WYSIWYG input field called `content`
- an image relational field called `image`
- a datetime selection field called `publish_date` - set the type to 'date'
- a many-to-one relational field called author with the related collection set to authors

In Access Control, give the Public role read access to the `authors`, `posts`, and `directus_files` collections.

Create 3 items in the posts collection - [here's some sample data](https://github.com/Marktawa/examples/tree/main/hugo-directus/demo-data/posts).

### Configure Hugo for fetching posts

Update `prebuild/layout/index.html` so that it can fetch posts from Directus as well. Append the following:

```html
{{ with resources.GetRemote (printf "%s/items/posts" site.Params.api_url) }}
    {{ $posts := unmarshal .Content }}
    {{ range $posts.data }}
        {{ $meta := dict "slug" .slug "title" .title "publish_date" .publish_date "author" .author.name "image" .image }} 
        {{ $metaJson := jsonify $meta }}
        {{ $content := .content | safeHTML }}
        {{ $output := printf "%s\n%s" $metaJson $content }}
        {{ $filename := printf "posts/%s.md" .slug }}
        {{ $resource := resources.FromString $filename $output }} 
        {{ $file := $resource.RelPermalink }} 
    {{ end }}
{{ end }}
```

Run `hugo` from the `prebuild` directory. This will generate a directory named `prebuild/public/posts` containing all the posts from the Directus collection.

Mount the `prebuild/public/posts` directory for Hugo to read the posts collection in `config.toml`.

```toml
[[module.mounts]]
    source = "prebuild/public/posts"
    target = "content/posts"
```

Create a layout for blog posts. Open the `layouts` directory and create a directory named `posts`. Inside `posts` add a file named `single.html`.

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">
    <title>{{ .Title }}</title>
</head>

<body>
    <img src="{{ printf "%/s/assets/%/s?download&width=600" site.Params.api_url .image }}">
    <h1>{{ .Title }}</h1>
    {{ .Content }}
</body>

</html>
```

Some key notes about the code snippet:
- The `download` attribute in the <img> tag is used to source the image from the Directus project.
- The `width` attribute demonstrates Directus' built in image transformations.

Run `hugo serve` and view any blog post page, for example http://localhost:1313/why-steampunk-rabbits-are-the-future-of-work.

![Single Blog Post]()

### Create Blog Post Listing

Create a new directory called `posts` in the `layouts` directory, and inside create `list.html` with the following content:

```html
<!DOCTYPE html>
<html>

<head>
    <meta charset="UTF-8">
    <title>Blog</title>
</head>

<body>
    <h1>Blog</h1>
    <ul>
    {{ range $posts.data }}
        <li>
            <a href="{{ printf "/posts/%s" .slug}}"><h2>{{ .title }}</h2></a>
            <span>{{ .publish_date }} &bull; {{ .author.name }}</span>
        </li>
    {{ end }}
    </ul>
</body>

</html>
```

This will display all a list of all the blog posts. It includes the `slug`, `title`, `publish_date` and `name` of the related `author`.

Visit http://localhost:1313/blog to view the blog post listing.

![Blog Post Listing]()

## Add Navigation

Add some navigation to your website by updating all the layout files with the following code just after the opening `<body>` tag:

```html
<nav>
    <Link href="/">Home</Link>
    <Link href="/about">About</Link>
    <Link href="/conduct">Code of Conduct</Link>
    <Link href="/privacy">Privacy Policy</Link>
    <Link href="/blog">Blog</Link>
</nav>
```

## Next Steps

Through this guide, you have set up a Hugo project and configured to query data from a Directus project. You have used a singleton collection for global metadata, dynamically created pages, as well as blog listing and post pages.

If you want to change what is user-accessible, consider setting up more restrictive roles and accessing only valid data at build-time.

If you want to build more complex dynamic pages made out of reusable components, check out our [recipe on doing just this](https://docs.directus.io/guides/headless-cms/reusable-components).

If you want to see the code for this project, you can find it [on GitHub](https://github.com/Marktawa/hugo-directus-blog).