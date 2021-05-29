# tejasn.com
Code for the personal blog hosted at https://tejasn.com.

## Development
Jekyll is used to generate the static website.

### CI
There is a GitHub action which builds the static website and publishes it to `gh-pages` branch whenever a commit is pushed to `main`

### Local Development
Make sure Jekyll and bundler is installed and then do
```
bundle install
bundle exec jekyll serve
```

## License
The contents of the site(articles, images, audio) are licensed as [Creative Commons CC BY 4.0](https://creativecommons.org/licenses/by/4.0/deed.ast) unless another license is explicitly specified on the individual page. The code used to build the site is licensed as
```
Copyright 2021 Tejas Nandanikar

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
```