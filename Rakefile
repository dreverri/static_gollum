require 'rubygems'
require 'rake'

require 'gollum'
require 'liquid'

# Commands to support
# preview (live display of templated site)
# generate (static files deployed to _site)
# clean (delete _site)

module Gollum
  class File
    attr_writer :version
    def populate(blob, path)
      @blob = blob
      @path = (path + '/' + blob.name)[1..-1]
      self
    end
  end

  class Wiki
    attr_reader :page_class
    attr_reader :file_class
  end

  class Page
    def to_liquid
      {"content" => formatted_data,
        "title" => title,
        "format" => format.to_s,
        "author" => version.author.name,
        "date" => version.authored_date.strftime("%Y-%m-%d %H:%M:%S")}
    end
  end
end

class StaticWiki
  def initialize(wiki)
    @wiki = wiki
    @templates = {}
  end

  def tree_list(commit, tree = commit.tree, sub_tree = nil)
    list = []
    path = tree.name ? "#{sub_tree}/#{tree.name}" : ''
    tree.contents.each do |item|
      case item
      when Grit::Blob
        if @wiki.page_class.valid_page_name?(item.name)
          page = @wiki.page_class.new(@wiki).populate(item, path)
          page.version = commit
          list << page
        else
          file = @wiki.file_class.new(@wiki).populate(item, path)
          file.version = commit
          if item.name == "layout.html"
            @templates[File.dirname(path)] = ::Liquid::Template.parse(file.raw_data)
          elsif path == "" && item.name == "Rakefile"
            # Skip Rakefile
          else
            list << file
          end
        end
      when Grit::Tree
        list.push *tree_list(commit, item, path)
      end
    end
    list
  end

  def generate(branch)
    unless File.exists? "_site"
      Dir.mkdir("_site")
    end

    commit = @wiki.repo.commit(branch)
    list = tree_list(commit)
    list.each do |page|
      if page.is_a?(Gollum::File)
        FileUtils.mkdir_p("_site/" + File.dirname(page.path))
        f = File.new("_site/" + page.path, "w")
        f.write(page.raw_data)
      end
      if page.is_a?(Gollum::Page)
        f = File.new("_site/" + page.name + ".html", "w")
        template = get_template(File.dirname(page.path))
        if template.nil?
          data = page.formatted_data
        else
          data = template.render( 'page' => page )
        end
        f.write(data)
        f.close
      end
    end
  end

  def get_template(path)
    if path == ""
      nil
    else
      @templates[path] or get_template(path.split("/")[0..-2].join("/"))
    end
  end
end

desc "Generates the static wiki"
task :generate do
  wiki = Gollum::Wiki.new("")
  static_wiki = StaticWiki.new(wiki)
  static_wiki.generate("master")
end
