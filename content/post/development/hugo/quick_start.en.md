---
title: "Quick start"
date: 2021-02-13T12:19:30+01:00
draft: false
weight: 5
tags:
  - "Hugo"

categories:
  - "development"
---

### How to create a new Hugo site?

1. Install Hugo following the [oficial docs](https://gohugo.io/getting-started/installing/).

2. Choose a theme [here](https://themes.gohugo.io/). For example, the theme of this website is [Learn](https://themes.gohugo.io/hugo-theme-learn/).

3. Create a new hugo project with:
```
hugo new site <name-project>
```

4. Init git repo.
```
git init
```
5. Copy the zip of the theme choosed in the new folder `themes` or add the submodule.
```
git clone https://github.com/matcornic/hugo-theme-learn themes/learn
```

Or add the submodule:
```
git submodule add https://github.com/gesquive/slate themes/slate
```
6. Edit configuration file `config.toml` and add:
```
baseURL = "http://localhost:1313/"
theme = "learn"
```

7. Run the website, it will be available at [http://localhost:1313/](http://localhost:1313/).
```
hugo server
```

