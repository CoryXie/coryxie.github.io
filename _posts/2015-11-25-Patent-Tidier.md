---
layout: post
title: "Introduction to Patent Tidier"
description: 
headline: 
modified: 2015-11-25
category: webappdev
tags: [WebApp]
imagefeature: 
mathjax: 
chart: 
comments: true
featured: true
---

There are a lot of ways for people to learn somthing, by reading books, reading thesis or papers, surfing the web with keywords for articles, subscribing channels with specific topics, etc. When it comes to technical knowledge, especially for some more formal details about implementation, I found reading patent documentation is quite helpful. The patent databases includes a lot (call it huge) of technical information in the past 100+ years, and we can safely say the patent databases form the history of modern tech industry. Although patents are mostly created/applied to prevent other parties from copying the ideas, in order to preserve its Return of Investment on the ideas (for money), if you take them the other way, by using it as source of learning, then no one can prevent you. This blog entry introduces the 1st Web App that I created for better Patent Reading.

# Problems of Patent Databases

There are a several web sites with patent databases, [Google Patents](https://www.google.com/?tbm=pts&gws_rd=ssl) and [Lens](https://www.lens.org/lens/) are the ones I recommend, although [USPTO](http://www.uspto.gov/) is also sometimes visited by me. However, a common problem of these databases is that the patent documentation is not friendly for reading. Oh, did you see the `Lens.org` search box in each post of this blogging site?

For example, I hope to read it like a book or web site articale, with the accompany images embedded just around the text describing the images. However, almost all patents are created with images separated from descriptions. Google Patents site is *kind enough* to provide these images (if avaiable) as a list of clickable thumbnails, allow people to enlarge the images to see details if they want to. However, asking the readers to go up and down, cliking here and there, is really annoying and distracting for knowledge learning purposes. Even if you printed out the patent PDF, you don't want to look back often while reading a piece of description with "refering to FIG.2", "block 202",..., etc. 

For another example, most of these patents readable from web sites are converted from original paper backed patent application documents with OCR scanners. This creates a lot of OCR issues, such as wrong texts (not correctly recognized by the OCR), or strange formulas that are hard to understand. Sometimes, you have to refer back to the picture or PDF version of the same document to make sure some descriptions are what you understand.

Sometimes people even want to save the patents for offline reading, especially with eBook readers such as Kindle. However, the PDF version of the patents are really hard to read on eBook readers, not to say you have to go back and forth with these images.

Sometimes people may even want to share the patents with friends, or discuss the contents with them. Most sites do not have such features to allow Social Networking. 

With all these problems, I still love to search patent databases to read for technical information. To make it better for reading through, I've created [Patent Tidier](https://github.com/CoryXie/PatentTidier).

# Patent Tidier Features

[Patent Tidier](https://github.com/CoryXie/PatentTidier) is an **Open Source Chrome Extension** with a good list of features.

* Displays an icon in the address bar to allow users to toggle visibility of unnecessary Google Patents sections.

<img src="{{ site.baseurl }}/images/2015-11-25-1/toggle-button.png" alt="Patent Tidier Toggle Button">

* Inserts accompany images into proper places in the patent description sections for better readability.
* Inserted patent images can be rotated to proper orientation by clicking on the images for better readability.

<img src="{{ site.baseurl }}/images/2015-11-25-1/insert-images.png" alt="Patent Tidier Insert Images">

* Load Patent PDF file along side the OCR generated patent descriptions for viewing side by side.

<img src="{{ site.baseurl }}/images/2015-11-25-1/pdf-side-by-side.png" alt="Patent Tidier PDF Side by Side">

* Allow users to edit (correct) the OCR generated patent descriptions by clicking on these descriptions.

<img src="{{ site.baseurl }}/images/2015-11-25-1/edit-desc.png" alt="Patent Tidier Edit Description">

* Allow users to save patent descriptions as Microsoft Word document (with accompany patent images). This feature needs a Chrome extension called 'Allow-Control-Allow-Origin: *' to overcome the "cross-origin resource sharing" problem for saving the patent images.

<img src="{{ site.baseurl }}/images/2015-11-25-1/save-word.png" alt="Patent Tidier Save Word & Send Blog">

* Allow users to send patent information to user configured email address (with a Blog entry added on [www.justsmart.mobi](www.justsmart.mobi)). Of course, this feature has to be expanded for actual usage.

<img src="{{ site.baseurl }}/images/2015-11-25-1/config-extension.png" alt="Patent Tidier Config Extension">

# Expected Features

* The emailing and blogging features may need to be expanded further, becasue I have only done all the above list in two weeks of offline hours, these are really some prototypes to born such features.
* Maybe I will need to spend some time generating the patent refernce history of a specific patent, then we can find where one idea is tracked back in the tech history.