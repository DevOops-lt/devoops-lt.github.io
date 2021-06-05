# DevOops.lt blog

## Setup

This Blog is hosted on Github Pages and is published automatically each time `master` branch is updated.

Blog engine is [`jekyll`](https://jekyllrb.com/docs/) and theme is [`minimal mistakes`](https://mmistakes.github.io/minimal-mistakes/).

Domain is bought from [domains.lt](https://domains.lt) and is managed in [CloudFlare](https://cloudflare.com)

## Contributing

### New Author

To setup a new author you need to edit `data/authors.yml` and add a new author as in the example below.

```yaml
# Block starts with an Author Name which is used for referencing and is not actually displayed
<AUTHOR NAME>:
  # This is the author display name
  name             : "<AUTHOR NAME>"
  # Please upload an avatar image to assets/images folder and reference it here
  avatar           :  "/assets/images/<AUTHOR IMAGE>.jpg"
  bio              : "<AUTHOR BIO>"
  location         : "<AUTHOR LOCATION>"
  email            : "<AUTHOR EMAIL>"
  # All supported links are below, all are optional and can be ommited, although generaly some information is desired.
  links:
    - label: "Email"
      icon: "fas fa-fw fa-envelope-square"
      url: "mailto:<AUTHOR EMAIL>"
    - label: "Website"
      icon: "fas fa-fw fa-link"
      url: "https://devoops.lt"
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      url: "https://github.com/<AUTHOR GITHUB NAME>}"
    - label: "Twitter"
      icon: "fab fa-fw fa-twitter-square"
      # url:
    - label: "Facebook"
      icon: "fab fa-fw fa-facebook-square"
      # url:
    - label: "GitHub"
      icon: "fab fa-fw fa-github"
      # url:
    - label: "GitLab"
      icon: "fab fa-fw fa-gitlab"
      # url:
    - label: "Bitbucket"
      icon: "fab fa-fw fa-bitbucket"
      # url:
    - label: "Instagram"
      icon: "fab fa-fw fa-instagram"
      # url:
```

Now when creating a new post you can reference the author by the block name of the author.

```yaml
---
layout: single
title:  "Terraform coding guidelines"
date:   2021-05-05 13:20:00 +0200
categories: Jekyll update
author: <AUTHOR NAME>
---
```

### New Post

All post are stored in `_posts` directory and are written in `markdown`, file names are expected to match `YEAR-MONTH-DAY-title.md` format.

All new posts have to start with metadata information to be correctly identified and processed.

```yaml
---
layout: single
title:  "Main Post Title"
date:   2021-05-05 13:20:00 +0200
categories: Jekyll update
author: <AUTHOR NAME>
---
```

* `single` layout is the most commonly used more about the supported layouts and their differences can be found [here](https://mmistakes.github.io/minimal-mistakes/docs/layouts/)
* Please keep the `date` format as specified above
* Categories are create automagically get creative :)
* If for some reason you cannot identify the author `Default Man` can be used which will update the post with general site information
