---
title: "My First PR: Lessons Learned"
date: 2026-04-23
---

Earlier today, I had my first ever pull request merged. For many, submitting PRs is entirely ordinary and mundane. For me, this was a significant milestone and the fulfillment of a childhood dream.

I have been using open source software for over a decade, and always dreamed of contributing to open source in some way. When I was a teenager, however, I assumed that all contributors were package maintainers or coding gurus. I've made several attempts over the years to learn how to program, but something in my brain just doesn't like it. I can grasp the basics, and I can cobble together basic automation scripts, but it just isn't as fun as movies make it look.

Instead, I went into humanities. Culture, language, literature, religion: these are the things that interest and excite me. But I still also love open source. I love the ethic of it: a decentralized, democratized, almost anarchic model for software development and distribution. In theory, anybody can contribute, anyone can benefit. It's what we 90s kids were promised technology would be; technology freed from the shackles of the [Peter Thiels](https://www.youtube.com/watch?v=Kl0FfBQgsB8) of the world.

## Obstacles

Open source is intimidating. Open source does not have a good reputation for friendliness, much less so toward newcomers. Code reviews are brutal; documentation is written with the assumption that you already know how to use git and read source code. Many projects operate on a model of "ask forgiveness, not permission", "the doers decide", "just jump in", or "read the fucking manual".

I've been daily-driving Linux for 15 years but I'm not a computer scientist. I've used git as a replacement for cloud storage and revision history, but until this week I had never created a branch. The prospect of "just jumping in" was impossibly daunting. Jump in where? And do what? How is a nontechnical contributor supposed to navigate repos without any guidance? It seems unfair to simply point at the docs, especially when the majority of projects are horribly documented. It doesn't always feel safe to just dive into a Matrix chat, especially when certain corners of the open source community have a reputation for sexism, racism, homophobia, etc.

I like structure. I like to know what people expect of me, where I am, where I'm going. It's a tall order to ask me to join a Matrix channel and say "Hi, I know absolutely nothing, please figure out how I can be useful." I've never had a job in sales. I hate the idea of cold calling. Every job I've ever had I've gotten by replying to a job posting and interviewing. I had no precedent for this social model.

Still, I don't like feeling like a freeloader. I never enjoyed being "in" the Linux ecosystem, enjoying the fruits of contributors' unpaid labor, and not chipping in. So I would regularly poke around, read forums and codes of conduct and Reddit threads, looking for a way in.

I hate capitalism and mistrust large institutions. Naturally, I tended to be drawn to the smaller, independent projects. I'm also a transgender Korean Jew. I'm no stranger to being treated... weirdly in online spaces, and the small projects often have the least ability to regulate that kind of behavior. Open source self-selects for self-motivated coders, and codes of conduct and governance are often afterthoughts: they're only implemented after problems occur. By then, it's too late, and the damage has been done.

Fedora Project was the last place I expected to find a home, but here I am. It helped that I stumbled on a recording of a [docs workshop](https://www.youtube.com/watch?v=F3CN_OxJp94) on YouTube, which showed me that there were at least some people in the community who were willing to take the time to walk newcomers through the process. It helped that the Join SIG has a semi-structured onboarding process with regular check-ins and a designated space where newcomers could safely ask stupid questions and a team to help them find their way.

So I wound up making a FAS account, introduced myself, said something along the lines of "I'm a library and information science student with an anthropology background, what can I do?" I was, to my surprise, very warmly greeted. Even more to my surprise, LIS is actually a useful field. It turns out LIS professionals make pretty good "data taxonomists" or "metadata specialists", and my weird combination of love-for-open-source and love-for-humanities puts me in a good position to make a decent technical writer. I was referred to the Data Working Group and was immediately put to work on the [Datanommer dictionary](https://codeberg.org/fedora-mwinters/datanommer-dictionary), which will eventually be put to use to analyze community health.

(Special thanks to @MatH for helping me find my way and to @mwinters for essentially becoming my *de facto* mentor.)

I love community health! This entire essay has been, in a way, *about* community health.

I've also popped into the DEI channel, which is a natural fit given the whole everything about me, and made some suggestions for the next Fedora Week of Diversity drawn from my experience doing oral history interviews last semester; and the Docs channel, which is really popping off lately.

## Mistakes Made

On to the PR itself.

It turns out that you can't just `git clone` somebody's repo, commit your changes, and `git push` to create a PR. No, you have to *fork* the repo first. Then, apparently, you're supposed to create a branch, and commit your changes there! Then you can push to your fork, and then you go to your web browser (there must be a way to do this with the cli but I sure don't know it yet) to submit the PR.

Then you wait...

Then you get an email notification and it turns out some of the topics you added to `topics.csv` are weird and don't actually exist in the wild even though they exist in the code, so you'd best please label those. You do that, but you accidentally create *another branch*, and when you push it doesn't update your PR because the commit is on an unrelated branch, so in a panic you delete the branch because you don't know how branches work really, and redo your fixes because you just deleted them like an idiot, and then finally correctly push the revisions and leave a comment on the PR saying you fixed it.

Then you wait...

And you put your little comments in the wrong field on the CSV because you forgot you were editing a CSV and thought you were writing JSON comments, oops, what a fucking idiot, gotta fix that and push it and comment on the PR...

And wait...

And then you get an email saying that your pull request was merged! And folks in the Matrix channel are congratulating you on your first successful PR!

It's a strange feeling, finally being a contributor and not just a user. I spent years assuming the door was closed to people like me, people who aren't particularly technical and don't know how to code, people who are artists, social scientists, philosophers, or anthropologists. It turns out the door was open. It just needed a sign.

I hope this essay is a sign for someone.
