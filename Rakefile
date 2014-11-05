require "kramdown"
require "coderay"
require "fileutils"

MARKDOWN_FILES = Dir.glob("#{__dir__}/articles/**/*.md")

task default: :html_files

desc "Generate HTML files from markdown articles"
task :html_files do
  MARKDOWN_FILES.each do |markdown_file|
    html_path = markdown_file.sub("/articles/", "/articles-html/").sub(/\.md$/, ".html")
    puts "Generating #{html_path}"
    FileUtils.mkdir_p(File.dirname(html_path))
    File.open(html_path, "w") do |html_file|
      filecontent = File.read(markdown_file)
      filecontent = filecontent.gsub("\`\`\`", "~~~")
      filecontent = Kramdown::Document.new(filecontent, template: "#{__dir__}/templates/default.html.erb")
      html_file.write(filecontent.to_html)
    end
  end
end

desc "Delete all generated HTML files"
task :clean do
  FileUtils.rm_rf("#{__dir__}/articles-html")
end
