---
title: "Focusing on Long-Term Value"
description: "With long-term value in mind, hackab.it is redesigned to optimize the reading experience."
tags: [jekyll, nico, long term, blog]
---

I've been a Stack Overflow [user](http://stackoverflow.com/users/1240620/kavinyao) for over 2 years. I love Stack Overflow because it's optimized for long-term value. Last year, I attended a [seminar](http://hci.stanford.edu/courses/cs547/speaker.php?date=2013-11-01) by Jeff Atwood, co-founder of Stack Exchange, a particular point he mentioned is to **maximize signal and minimize noise at design level**. That's the reason why Stack Overflow doesn't become a developer forum, why the content is always of high quality and free from trolling, and why you can always find the right answer to your problem on it.

I'm not an active user, but every time I think I can contribute a better answer, I do. Most of my answers are late to the original question. Nevertheless they get up-voted. It's always a joy for me to receive a notification that my answer gets up-voted.

**Time isn't a big matter if your site is optimized for the long term.**

I started this blog two and a half years ago. In [Why Blog?](http://hackab.it/2012/01/why-blog/) (in Chinese), one of the initial posts, focusing on long-term value is one of the goals I set for this blog. Till now, I wrote 18 posts. On Google Analytics, I see people come to my site for particular technical problems. A few days ago, I even discovered that 2 answers on Stack Overflow refer to a blog post on this blog.

These facts show that this blog is building the long-term value. This makes me proud.

Two days ago, I switched the blog engine from [nico](https://github.com/lepture/nico) to [jekyll](http://jekyllrb.com/). What you are looking at is version 4.0 of my blog. I made this transition for two major reasons:

**Reason 1: I'm not satisfied with the previous design.** I always care about the reading experience. I admire blogs with optimized reading experience: no distractive elements, content only and beautiful font. I believe that if your blog lacks great reading experience, the value of it discounts.

I designed [the previous theme](https://github.com/kavinyao/nico-minimalist) with minimalism in mind: minimal elements, content only, no external resources. But gradually I realized that I didn't get it right. Such restriction is what prevents the blog from being great. For example, I need a great typeface optimized for reading, with consistent experience across different platforms. Using system fonts only is not enough.

As a result, a new design of the blog has been hovering in my head for a long time. Luckily, jekyll comes with a clean, responsive theme, which is a great starting point. With [a few tweaks](https://github.com/kavinyao/hackab.it/commits/master/css/main.css), I have a blog optimized for reading experience.

**Reason 2: jekyll fits my needs for a static site generator.** With the recent release of 2.0, jekyll has builtin Sass and CoffeeScript support and I love both Sass and CoffeeScript. You can use [data files](http://jekyllrb.com/docs/datafiles/) to populate data into your site. With data files, you can separate data from template. Liquid template reminds me of Django but is more restrictive. (Frankly, I miss jade for its concise syntax. However, the ability to execute arbitrary code in template, to me, is not a good idea.)

As a programmer, I believe **every tool you use should be programmable**. The best part is that besides all the builtin features, jekyll is fully customizable and extensible, with excellent documentation. Using jekyll also helps me learn Ruby.

A blog can never be great without great content. With the right toolset provided by jekyll, my blogging experience would be more enjoyable.

As always, this transition doesn't break URL. Full-text RSS output and Twitter card support are enabled as well.

Hope you like the new design.