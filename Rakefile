require "date"
task :new_post, [:title,:date,:link] do |t,args|
  date = if args[:date].nil? || args[:date] == "today"
           Date.today
         elsif args[:date] == 'tomorrow'
           Date.today + 1
         else
           Date.parse(args[:date])
         end

  title = args[:title]

  formatted_date = date.strftime('%Y-%m-%d')

  slug = formatted_date + "-" + title.gsub(/[\s\W]/,"-").downcase

  post_filename =  "_posts/#{slug}.md"

  File.open(post_filename,"w") do |file|
    file.puts "---"
    file.puts "layout: post"
    file.puts "title: \"#{title}\""
    file.puts "date: #{formatted_date} 9:00"
    if args[:link]
      file.puts "link: #{args[:link]}"
    else
      file.puts "ad:"
      file.puts "  title: \"Focus on Results\""
      file.puts "  subtitle: \"11 Practices You Can Start Doing Now\""
      file.puts "  link: \"http://bit.ly/dcsweng\""
      file.puts "  image: \"/images/sweng-cover.png\""
      file.puts "  cta: \"Buy Now $25\""
    end
    file.puts "---"
    file.puts
    if args[:link]
      file.puts "This is the intro and a reference to [the link][link], as well as a quote:"
      file.puts
      file.puts "> this is a quote from the articel"
      file.puts
      file.puts "And a follow up quip if needed"
      file.puts
      file.puts "[link]: #{args[:link]}"
    else
      file.puts "This is the intro"
      file.puts
      file.puts "<!-- more -->"
      file.puts
      file.puts "This is some content"
      file.puts
      file.puts "<div data-ad></div>"
      file.puts
      file.puts "This is some more content"
    end
  end
  puts post_filename
end
