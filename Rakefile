desc "Create New post"
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
