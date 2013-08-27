---
layout: post
title:  "Don't be too expressive"
date:   2013-08-26 11:07:00
---

Naming things in code is hard. We programmers know that. So we came up with some rules of thumb to make our code more expressive. For example, we all know it's bad to name a method `xpvha()`. It doesn't give a single clue of what that method does. So we better use methods with longer names, correct?

I'd like to point out is that identifiers can be **too expressive**. `user_with_expired_password_or_with_more_than_three_invalid_login_attemps` is just as bad. And it's bad because it's a definition - long, requires a high cognitive load to understand. Our brains will automatically struggle to come up with and abstraction so it doesn't have to remember all of that, which is something the programmer should have done in the first place.

So why we don't save our code readers (which can include my future self) the trouble and do it ourselves?

{% highlight ruby %}
  def locked_user?
    password_expired? || login_attemps.invalid.count > 3
  end
{% endhighlight %}

So if I'm reading the code and I know what a *locked user* is, great. If I don't, then I'll go looking for a definition (bonus points if your method body does that clearly). If you don't know the words, go ask someone. I can bet it's in the business domain language.

Using too long method names is just as bad as using too short ones. Be expressive but keep it concise.

Comments on this article available on [hacker news](https://news.ycombinator.com/item?id=6280647).