---
layout: post
title: 42 Key, Split Keyboard Setup
description: >
  Initial setup of a 42 key ergonomic, split keyboard.
# image:
#   path: /assets/img/blog/jupyter.jpg

categories: [Setup, Hardware]
tags: []
---

Working in 21st century has exchanged manual work for "knowledge work". Whilst that
means that people are not breaking their back in the field anymore, they are suffering from other types of injuries.
Out of these, most common are due to the lack of physical activities. Yeah ... sitting at the PC
for 8 hours a day turns out to not be so healthy. Getting the basics done, by working
out and taking care of the body, shoud come as a priority. After the basics are fulfilled,
there are other parts that one could do to make the workflow more optimal and minimize
more subtle injuries.

Optimality is extremely constrained to what a person is doing. If most of the time
a person spends is not in front of the PC then it makes sense to not invest in
making that workflow optimal. By the same token, a writter has a different requirements
for his work than a programmer or say ... a video editor.

As part of my work, I am constantly writing code. Be it for my PhD or my personal
projects. My editor of choice is NeoVim, which means I've ditched the mouse and flexed
to every person I know. Just kidding, or am I? Either way, this does mean that I am
spending a lot of time using the keyboard and using symbols and shortcuts which lead me
to an understanding of the notorious "emacs pinky", or at least a subtle distaste
for pressing these "Ctr+" and "Shift+" combos (and there are a lot of them).

So what are the alternatives? Personally, I do not see many, besides the use of
a custom keyboard that allows you to set things as you wish. This way, you can make it
adapt to your workflow. As for me, since I already decided to get a custom keyboard,
I also decided to go for a split keyboard. And why stop there and not just constrain myself
to only using 42 keys!? So that's exactly what I did.

## 1. Hardware

Choosing the keyboard itself proved to be a difficult endeavour giving that there are numerous
alternatives and most of them are quite pricey. In the end, I settled for a [42 keys keyboard](https://www.etsy.com/uk/listing/1571855869/choc-corne-40-24g-wireless-split?ga_order=most_relevant&ga_search_type=all&ga_view_type=gallery&ga_search_query=split+keyboard&ref=sr_gallery-1-2&content_source=437bcaab3561e847c8039d430bff6e6bb4ef4c31%253A1571855869&organic_search_click=1).
What attracted me, besides the price, was the long lasting battery life and the dongle that
allows me to use it at boot time.

As the keyboard comes with no keyswitches and keycaps, I acquired these from [splitkb]().
I went with the red pro switches, [link here](https://splitkb.com/collections/switches-and-keycaps/products/kailh-low-profile-choc-switches), with [MoErgo POM MBK-Profile Keycaps](https://splitkb.com/collections/switches-and-keycaps/products/moergo-pom-mbk-profile-keycaps).

## 2. Software

The keyboard comes pre-installed with [vial](), a software built on top of [QMK]() that allows
for real time changes (amazing for people that are not decided and hence, me).

## 3. Keymaps

### 3.1. Layout

The first thing to decide is weather or not to keep the layout. There are a few layouts
that stand out, such as [devorak](), [colemak]() and [workman](). They all improve the
efficiency compared to [querty](). Ultimately, I decided to not incapacitate my ability
to work with other keyboards and went for _querty_.

### 3.2. Symbols Layer

As a programmer, I was the most excited about the symbols layer. For me, this meant that
I could program longer and have a better time doing so. For this, one of the biggest sources
of inspiration was the blog by Pascal, [Designing a Symbol Layer](https://getreuer.info/posts/keyboards/symbol-layer/index.html). I took the lazy approach
and used the symbol layer proposed by him.

### 3.3. Navigation/Numbers Layer

This layer took most time tinkering with. I started with the following constraint, I wanted
to have my navigation on HJKL, so that they are directly related to VIM keybinds. I decided
to put my number layer on the left and have a numpad style, this proved annoying. After few
trials, I decided to set my keyboard as above. This way I had access to the frequent (0,1) on
my index finger.
