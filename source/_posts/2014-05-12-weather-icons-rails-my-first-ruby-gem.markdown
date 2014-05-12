---
layout: post
title: "Weather-Icons-Rails: My First Ruby Gem"
date: 2014-05-12 11:58:32 -0700
comments: true
categories: [CSS, SASS, Ruby On Rails, Ruby, Ruby Gems]
---

First I want to give a huge thanks to [Lukas Bischoff](http://www.twitter.com/artill)
for designing this font, and [Erik Flowers](http://www.helloerik.com/) who
originally added the css to turn it into a web font, and finally the
[Font Awesome Sass](https://github.com/FortAwesome/font-awesome-sass) gem that
I used as a template for [Weather Icons Rails](http://rubygems.org/gems/weather-icons-rails).

I originally started this project because of the lack of an easy to use web
based font that included weather icons. I had been working on my current
project [EarlyWord](http://earlyword.herokuapp.com/), and was hoping to use
Font Awesome to add icons to my weather app. Sadly the only weather font they
had was the download from the cloud icon. This wasn't going to be of any use if I
hoped to give the user an immediate sense if the weather through a small image.
We wouldn't want users to think global warming had begun to cause data to fall
from the skies...I was then fortunate enough to find Erik Flowers work, which is
composed of some really *Awesome* fonts.

From here I ran into two problems. One was that my project didn't use Less, but
instead used Sass. This was easily solvable by converting it to pure CSS and just
including that into the assets file. This prevented me from using any dynamic
coloring, and would make adding any special calculations for affects on the
icons difficult. The second problem was the much larger amount of code that was needed
to support pure CSS, especially once all of the mix-ins were removed and all the dynamic
variables set to their static values. Instead I decided to convert it into Sass
to work with the rest of my project.

At this point I got carried away and decided to check out Font Awesome Sass,
and started working on a replica of it using these new fonts. It was surprisingly
simple, especially once I discovered all of the commands rake provides to help with
gem manufacture. I added a bunch of ruby helper methods, and modularized the CSS into
sperate files. Using Sass I was able to create much more maintainable code using
variables instead of hard coding all the CSS class prefixes.

After converting everything to Sass, and still not being completed satisfied with
all of the code that now cluttered my assets folder, I packaged it into its own
gem. Now I hope that others will find it as useful as I did. I would love to hear
any suggestions on improving it. Still a work in progress, and will hopefully
finish the test suit next. You can download the gem at
[Weather Icons Rails](http://rubygems.org/gems/weather-icons-rails),
and see the source code over on [GitHub](https://github.com/CodySchaaf/weather-icons-rails).
