# Ruby

- gem: ruby package 
    - jekyll is gem 
    - other jekyll plugins are gems 
- gemfile: list of gems used by site 
- bundler: gem that installs all files in `Gemfile`
    - to install gems in gemfile using bundle: `bundle install`, `bundle exec jekyll serve`
    - or can just run `jekyll serve` if not using a Gemfile


---
# Jekyll

- Jekyll: static site generator, so buidls site before we view 
    - `jekyll build`: build, output statie site to directory called `_site`
    - `jekyll serve`: does `jekyll build`, runs it on local web server
    - `jekyll serve --livereload`: force browser to refresh with every change

## Liquid

- Liquid: templating language w/ 3 main components: 
    - objects: use to show variable values on page (ex. `{{ page.title }}`)
    - tags: define logic + control flow for templates: 
ex. 
```
{% if page.show_sidebar %}
    <div class="sidebar">
        sidebar content 
    </div>
{% endif %}
```

    - filters: change output of a Liquid object (ex. `{{ "hi" | capitalize }}` 

---

- Jekyll supports markdown + HTML
- layouts: templates that can be used by any page in your site and wrap around page content (stored in `_layouts` idrectory)
- `include` tags lets you include content from another file stored in `_includes` folder (good for shared source code around the site)
- load data from YAML, JSON, CSV files located in `_data` directory, used to separate content from source code (ex. use case: all the blogs, etc.)
- store assets in `assets` folder with `css`, `images`, `js` subfolders
- can also have `_sass` folder: Sass (CSS extension) is baked into Jekyll

---
## Blogging in Jekyll 
- blogging is powered by only text files (blog posts live in `_posts` directory)- posts filename has a special format
- you can list all posts w/ `site.posts`

- **collections**: similar to posts, except the content does not have to be grouped by date
- can set up collections by modifying Jekyll configuraiton in `_config.yml`
- documents (items in the collection) live in a folder in the root of the site named `_*collection_name*`

---

# My Website Brainstorm: 

- have `blog` as its collection ? 
- have `ctf-writeups/web?` `ctf-writeups/misc` (separate by category)? as its collection ?

- have base with the fishy text at bottom?
- https://jun711.github.io/web/how-to-highlight-code-on-a-Jekyll-site-syntax-highlighting/ -> add this 

- how to have collapisable sections as needed? (also have to get the code for that in my older commits)
- fix aesthetic in page.html 

- have page.html template 
- I think the colors look different on the mac vs thinkpad bc of the DarkTheme extension - fix and figure out 

- make the left box say blog/recent.md or smthing
- say currently learning web! in description

- use blue for the titles and shit in the website instead, more peaceful

- need to make link that shows all of a section too

- check over all HTML, get rid of any boof shit
- fix the ugly scroll bar in mac

- fix purple/black styling in the pages -> have different colors for different collections?
- make the web/ lead to the web page
- create a custom 404 page 

- add the [pgp key], etc. popups
- have a yaml file list for the `/blog/recent`?

- change dates on website to render as default format for Linux
- fix navigation color links + when you click on them, nt showing ocrrectly 
- have a 'dir' page for 'ctf-writeups'?  

- this URL doesnt make sense w/ the path:http://localhost:4000/ctf-writeups-web/
- before publishing, figure out if the syntax highligting is correct 
- before publishing website, organize css

- can use this for a reference on gfm admonitions: https://github.com/Helveg/jekyll-gfm-admonitions
