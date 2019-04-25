---
layout: post
title: "Jekyll 自动生成文章"
category: Jekyll
tags: [Jekyll]
---

当使用Jekyll写文章的时候，你肯定不想麻烦的创建文本，修改文本后缀名，再加文本头加上yml语法开头。所以这时候你肯定想到的是写个脚本简化操作，程序员不就是为偷懒而写代码吗？可以使用Rake来解决这个问题。

### Rake

Rake ，即Ruby Make ，使用ruby开发代码构建工具

1. 安装rake

		gem install rake
		#可以先查看是否已经安装rake
		gem list rake 
		
2. 编写Rakefile，放入jekyll的根目录下

		require 'rake'
		require 'yaml'
		
		SOURCE = "."
		CONFIG = {
		  'posts' => File.join(SOURCE, "_posts"),
		  'post_ext' => "md",
		}
		
		# Usage: rake post title="A Title"
		desc "Begin a new post in #{CONFIG['posts']}"
		task :post do
		  abort("rake aborted: '#{CONFIG['posts']}' directory not found.") unless FileTest.directory?(CONFIG['posts'])
		  title = ENV["title"] || "new-post"
		  slug = title.downcase.strip.gsub(' ', '-').gsub(/[^\w-]/, '')
		  filename = File.join(CONFIG['posts'], "#{Time.now.strftime('%Y-%m-%d')}-#{slug}.#{CONFIG['post_ext']}")
		  if File.exist?(filename)
		    abort("rake aborted!") if ask("#{filename} already exists. Do you want to overwrite?", ['y', 'n']) == 'n'
		  end
		
		  puts "Creating new post: #{filename}"
		  open(filename, 'w') do |post|
		    post.puts "---"
		    post.puts "layout: post"
		    post.puts "title: \"#{title.gsub(/-/,' ')}\""
		    post.puts "date: #{Time.now.strftime('%Y-%m-%d %H:%M:%S')} +0800"
		    post.puts "category: "
		    post.puts "tags: []"
		    post.puts "---"
		    post.puts "* content"
		    post.puts "{:toc}"
		  end
		end # task :post


这是一个简洁的版本，你也可以自行添加你的description，categories，tag等。

命令行输入 `rake post title="article name"`。 马上会在_post创建年-月-日-hello-world.md文章。 

	$ rake post title="rake"
	Creating new post: ./_posts/2018-09-25-rake.md
