desc "Create new post file"
task :post do
   print "Post title: "
   title = STDIN.gets.strip
   name = "_posts/" + Time.new.strftime("%Y-%m-%d-") + title.downcase.gsub(" ", "-") + ".markdown"
   if title and not File.exists?(name)
       File.open(name, "w") {|file|
           file.write("---\nlayout: post\ntitle: #{title}\n---\n")
           file.flush
       }
    puts "#{name} created"
    end
end


desc "Create new project file"
task :proj do
   print "Project title: "
   title = STDIN.gets.strip
   folder = "projects/" + title.downcase.gsub(" ", "-")
   if title and not File.exists?(folder)
       Dir.mkdir(folder)
       File.open(folder + "/index.html", "w") {|file|
           file.write("---\nlayout: page\ntitle: #{title}\n---\n")
           file.flush
       }
    puts "#{folder} created"
    end
end

desc "Create new static page"
task :page do
   print "Page name: "
   title = STDIN.gets.strip.downcase.gsub(" ", "-")
   if title and not File.exists?(title)
       Dir.mkdir(title)
       File.open(title + "/index.html", "w") {|file|
           file.write("---\nlayout: page\ntitle: #{title}\n---\n")
           file.flush
       }
    puts "#{title} created"
    end
end
