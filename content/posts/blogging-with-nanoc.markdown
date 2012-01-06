---
kind: article
created_at: 2011-12-23
title: Blogging With Nanoc

---
I've finally made the move to a static blog engine and I am using [Nanoc][nanoc].
I chose Nanoc because it met 90% of my requirements and adding the last 10% was
trivial.

My requirements for this site was to create all posts in [Pandoc][pandoc] flavored
markdown, use HAML for layouts and other pages, and use [Pygments][pygments]
for syntax highlighting in posts. I also did not want to be forced to use a certain
URL or file convention and wanted to develop my own for the site. Nanoc ships
with a very simple [DSL][dsl] that makes achieving those requirements trivial.


## Creating Rules ##
Nanoc, at its very core is a very simple compiler that follows to rules defined
in a `Rules` file. The `Rules` file is a ruby file that uses Nanocs DSL to define
`compile` and `route` rules. These rules define transformations on input content
and the filenames of the output files respectively. Each rule operates on an item.
An item in Nanoc is any type of input file, for example it can be a markdown file, an image
or CoffeeScript file.

## Compile Rules ##
The `compile` rule defines transformations by invoking filters which do the actual
transformations on the input file. Nanoc ships with [many filters][filters-list] out of the box
including a HAML filter, LESS filter and an ERB filter. It is also very easy
to write your [own filters][own-filters] that operate on other kinds of input.

Lets look at the `compile` rule that creates the posts on this blog:

~~~~~~~~~~~~~~~~~~~~~~~~ {.ruby}
compile '/posts/*/' do
  ext = item[:extension].nil? ? nil : item[:extension].split('.').last
  if ext == 'markdown' || ext == 'pandoc'
    filter :pandoc
    filter :pygments
  else
    raise "Unknown ext: #{ext} with item: #{item.attributes}"
  end

  layout 'default'
end
~~~~~~~~~~~~~~~~~~~~~~~~

This rule operates on all the files that are in the `/posts/` directory of my
input files. Then, for each item I pass it through two filters. A
`pandoc` filter and a `pygments` filter. The first transforms the post in to HTML,
and the second adds syntax highlighting to the code bocks in the post. At the end
I just apply the `default` layout to the item. This produces the final HTML of the
page.

In this rule I could have applied as many filters as I wanted or needed. I could
have also applied them conditionally as well based on some metadata that I added
to the item. The beauty of nanoc is that filters are very simple to use and chain
and I also have the full power of Ruby at my fingertips so I can make whatever
transformations that are required.

## Filters ##

In the above rule I used two filters that did not ship with Nanoc. I had to create
them because Nanoc does not ship with a Pandoc filter and I had to then create my
own syntax highlighting filter because I could not get the builtin one to work.
Lets look at the `pandoc` filter first:

~~~~~~~~~~~~~~~~~~~~~~~~ {.ruby}
require 'pandoc-ruby'

class PandocFilter < Nanoc3::Filter
  identifier :pandoc
  type :text

  def run(content, params = {})
    ::PandocRuby.convert(content, 'smart')
  end
end
~~~~~~~~~~~~~~~~~~~~~~~~

Nanoc filters are *stupidly* easy to write. All you need to do is create a class
that inherits from `Nanoc3::Filter`, give it a unique identifier and define a
`run` method which takes in a strong of content for you to transform. You just
need to return the result of the transformation. To write my `pandoc` filter
I used the wonderful [pandoc-ruby][pandoc-ruby] gem which wraps around the `pandoc`
executable.

Of course not all filters are going to be this simple or easier to write. Adding
syntax highlighting was significantly harder. My `pygments` filter at the time
of writing is below:

~~~~~~~~~~~~~~~~~~~~~~~~ {.ruby}
require 'pygments.rb'
require 'hpricot'
require 'cgi'

class PygmentsFilter < Nanoc3::Filter
  identifier :pygments
  type :text

  def run(content, params = {})
    post = Hpricot(content)
    code_blocks = post.search('pre.ruby code')
    code_blocks.each do |code_block|
      code = code_block.inner_html
      code = CGI.unescapeHTML(code)
      code = ::Pygments.highlight(code, :lexer => 'ruby', :options => {:lineseparator => '<br>', :encoding => 'utf-8'})
      code_block.parent.swap code
    end

    post.to_html

  end
end
~~~~~~~~~~~~~~~~~~~~~~~~

This filter assumes that the input is produced by my `pandoc` filter. It uses the
wonderful `Hpricot` library to parse the input. It then takes the content of the
code blocks created by Pandoc and sends it to Pygments. The resulting output replaces
the block created by Pandoc. The above code only highlights
Ruby, but it is trivial to extend to all the other programing languages that Pygments
supports.


## Routing Rules ##
After compiling items the next rule applied is the `route` rules.
The `route` rules define where an item should be placed in the output directory
and what filename it should have. Since the `Rules` file is pure Ruby you can leverage
all of ruby to create the URL scheme required. Here is the `route` rule for the posts on this site.

~~~~~~~~~~~~~~~~~~~~~~~~ {.ruby}
route '/posts/*/' do
  date = item[:created_at]
  raise "No Posted Date!" if date.nil?

  slug = item[:title].to_slug

  "/posts/#{date.year}/#{date.month}/#{date.day}/#{slug}/index.html"

end
~~~~~~~~~~~~~~~~~~~~~~~~

This rule simply returns the desired path and filename of the item in the output
directory. Here I use the [to_slug][to_slug] gem to help me generate a slug in the URL
from the post title.

*Like this post? Follow me on [Twitter][twitter].*

[nanoc]: http://nanoc.stoneship.org/
[dsl]: http://nanoc.stoneship.org/docs/api/3.2/Nanoc3/CompilerDSL.html
[pandoc]: http://johnmacfarlane.net/pandoc/
[pygments]: http://pygments.org/
[pandoc-ruby]: https://github.com/alphabetum/pandoc-ruby
[filters-list]: http://nanoc.stoneship.org/docs/4-basic-concepts/#filters
[own-filter]: http://nanoc.stoneship.org/docs/5-advanced-concepts/#writing-filters
[twitter]: http://www.twitter.com/zmanji
[to_slug]: http://rubygems.org/gems/to_slug
