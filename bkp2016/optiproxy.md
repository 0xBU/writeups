# OptiProxy
*(web, solved by 115)*  
>  inlining proxy here http://optiproxy.bostonkey.party:5300

This was a fun challenge about a vulnerable Ruby program. Navigating to the webpage given brings us to a 
simple website that prompts us:

```
welcome to the automatic resource inliner, we inline all images go to /example.com to get an inlined 
version of example.com flag is in /flag source is in /source
```

Unfortunately the flag is not as easy as just navigating to http://optiproxy.bostonkey.party:5300/flag.

Navigating to `/source` gives us the source code:

```
require 'nokogiri' 
require 'open-uri' 
require 'sinatra' 
require 'shellwords' 
require 'base64' 
require 'fileutils' 

set :bind, "0.0.0.0" 
set :port, 5300 
cdir = Dir.pwd 

get '/' do str = "welcome to the automatic resource inliner, we inline all images" str << " go to 
/example.com to get an inlined version of example.com" str << " flag 
is in /flag" str << " source is in /source" str end 

get '/source' do IO.read "/home/optiproxy/optiproxy.rb" end 

get '/flag' do str = "I mean, /flag on the file system... If you're looking here, I question" str << " your 
skills" str end 

get '/:url' 
do 
	url = params[:url] 
	main_dir = Dir.pwd 
	temp_dir = "" 
	dir = Dir.mktmpdir "inliner" Dir.chdir dir 
	temp_dir = dir 
	exec = "timeout 2 wget -T 2 --page-requisites #{Shellwords.shellescape url}" `#{exec}` 
	my_dir = Dir.glob ("**/") Dir.chdir my_dir[0] 
	index_file = "index.html" 
	html_file = IO.read index_file 
	doc = Nokogiri::HTML(open(index_file)) 
	doc.xpath('//img').each 
	do 
		|img| header = img.xpath('preceding::h2[1]').text 
		image = img['src'] 
		img_data = "" 
		uri_scheme = URI(image).scheme 
			begin if (uri_scheme == "http" or uri_scheme == "https") 
				url = image 
			else 
				url = "http://#{url}/#{image}" 
			end 
		img_data = open(url).read 
		b64d = "data:image/png;base64," + Base64.strict_encode64(img_data) 
		img['src'] = b64d rescue # gotta catch 'em all puts "lole" next end 
	end 
	puts dir FileUtils.rm_rf dir Dir.chdir main_dir doc.to_html 
end
```

At first glance, you might think to try exploiting `exec = "timeout 2 wget -T 2 --page-requisites #{Shellwords.shellescape url}`, but I was unable to find anything online that is able to escape 
`Shellwords.shellescape`. I believe it's safe to assume, that's a well written module, with no known 
exploits, and finding a 0day in it, while possible, is probably outside the scope of the challenge.

Moving on to understanding what exactly the code is doing, it seems to follow these steps:

1. Parse out the parameter given after `/`, e.g. `example.com` if navigated to 
`http://optiproxy.bostonkey.party:5300/example.com`
2. Create a folder named `inliner` and change directory into it.
3. Run `wget` to create a mirror of the website provided in (1)
4. Parse `index.html` from the mirrored website, and for each `img` tag:   
  1. Grab the `src` parameter of the `img` tag
     * If the `src` parameter's URI scheme is `http` or `https`, it's good to go
     * Else if the URI scheme isn't `http` or `https`, then form a string that does have `http` uri scheme
  2. Open the `url` string created in (4a), and read the data from it
  3. Base64 the image data read, and change the `src` parameter of the img tag to the base64 of the image 
data
5. Return the `inliner` dir to the user of the web app, and clean up

Okay, so where's the exploit? First things first, I tried passing something more interesting than just a 
simple webpage as the `url` to the web app to inline, i.e. `file://../../../../flag`, meaning I navigated 
to `http://optiproxy.bostonkey.party:5300/file://../../../../flag`, but that just returns to us the error 
message saying flag isn't there. I then thought that what I actually need is a website with an `img` tag 
whose `src` parameter is `file://../../../../flag`, so I whipped that up quickly:

```
<html><body> 
<img src="file://../../../../flag" />
</body></html>
```

I served the webpage off of my remote server with `python3 -m http.server` for a quick HTTP server. 
Navigating over to `http://optiproxy.bostonkey.party:5300/mywebsite.com:8000` gives me a blank screen. 
Checking the console and source code reveals the issue: `Not allowed to load local resource: 
file://../flag`

It looks like the web app did mirror my website, it just didn't actually open the file, because of the 
`uri_scheme` check, `file != http`. Okay, so we really do need something that'll pass the URI scheme check, 
back to Google to learn more about Ruby's `uri_scheme` leads me to 
http://sakurity.com/blog/2015/02/28/openuri.html, *jackpot*. Interesting, it seems that a URL like 
`http:/foobar` will pass the `uri_scheme` check, and if we can somehow have a folder named `http:` (note 
the `:`) then we have a remote file read (into `/flag`!). *Now... how do we possibly make the folder named 
`http:`.* 

After some thinking, I realized the solution, just make our fake website have a subdirectory named `http:` 
that'll get included by the `index.html`, because the `wget` call in the webapp is 2 levels deep, this will 
work! 

```
<html><body> 
<img src="./http:/hi" />
<img src="http:/../../../../../../../../flag" /> 
</body></html>
```

Note that the 2nd `src` tag above, is to `http:/`, and not `http://`. Getting that website inlined, will 
cause the Ruby web application to create a directory structure with a folder named `http:` in it, and then 
open a file in reference to the folder `http:`, and return to us:

```
<html><body> 
<img src="data:image/png;base64,aGloaQo=">
<img src="data:image/png;base64,QktQQ1RGe21heWJlIGRvbnQgaW5saW5lIGV2ZXJ5dGhpbmcufQo="> 
</body></html>
```

Decode the 2nd base64:
`BKPCTF{maybe dont inline everything.}`
