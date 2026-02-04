---
title: "Gitlab pages"
date: 2021-02-15T13:00:14+01:00
draft: false
weight: 15

tags:
  - "GitLab"
  - "CI/CD"

categories:
  - "devops"
  - "development"

---

### How to deploy in GitLab pages?

1. Set up CI/CD adding a file `.gitlab-ci.yml` with Hugo template. Change the master branch to main in the template.

```yaml
# This file is a template, and might need editing before it works on your project.
---
# All available Hugo versions are listed here:
# https://gitlab.com/pages/hugo/container_registry
image: registry.gitlab.com/pages/hugo:latest

variables:
  GIT_SUBMODULE_STRATEGY: recursive

test:
  script:
    - hugo
  except:
    - main

pages:
  script:
    - hugo
  artifacts:
    paths:
      - public
  only:
    - main
```

2. Modify the baseurl parameter with this structure `baseURL = "https://<gitlab-user>.gitlab.io/<project-name>/"`.

3. Enable the [Pages Access Control ](https://docs.gitlab.com/ee/user/project/pages/pages_access_control.html) for everyone visibility. Navigate to your project’s Settings > General and expand Visibility, project features, permissions > Pages > Everyone.