> This project have ceased to maintenance, welcome to try the new theme [hexo-theme-apollo](https://github.com/pinggod/hexo-theme-apollo).

![hexo-theme-jekyll-screenshot](https://cloud.githubusercontent.com/assets/9530963/10627194/0ceeb9bc-77e9-11e5-918b-c978d444bbd3.png)

## Install

``` bash
$ hexo init Blog && cd Blog && npm install
$ npm install --save hexo-renderer-jade hexo-generator-feed
$ git clone https://github.com/pinggod/hexo-theme-jekyll.git themes/jekyll
```

## Enable

Modify `theme` setting in site's `_config.yml` to `jekyll`:

```yaml
theme: jekyll
```

And add some related settings in the site's `_config.yml` as well:

```yaml
feed:
  type: atom
  path: /atom.xml
  limit: 20

jekyll:
  project: false
  selfIntro: true
  sign_image: image/bear.svg

# Social
social:
  github: https://github.com/xxxxxx
  coding: https://coding.net/u/xxxxxx
  weibo: http://weibo.com/xxxxxx

# Analytics
google_analytics: UA-80781234-1
baidu_analytics: //hm.baidu.com/hm.js?ee75cf111111aa99f8540efa2570970
```

* `feed` key is for the RSS/Atom generator, a feed generator plugin should be installed, such as `hexo-generator-feed`;
* `jekyll` is to control what sections to show on the index page, `sign_image` is the sign image link;
* `social` contains a list of social network links;
* `google_analytics` and `baidu_analytics` are for analytics script code for Google Analytics and Baidu Analytics.

## Add Demo.md

For better experience, you can remove default demo markdown file by using the follow command and add another markdown file provided by this theme:

```bash
$ rm source/_posts/hello-world.md && mv themes/jekyll/.post/demo.md source/_posts
```

## Run

```bash
$ hexo g && hexo s
```



## License

MIT
