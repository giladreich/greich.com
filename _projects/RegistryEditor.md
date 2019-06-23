---
layout: project
title: RegistryEditor
date: 4 Nov 2018
screenshot:
  src: /assets/img/projects/RegistryEditor/srcset@0,25x.jpg
  srcset:
    1920w: /assets/img/projects/RegistryEditor/srcset@1x.jpg
    960w: /assets/img/projects/RegistryEditor/srcset@0,5x.jpg
    480w: /assets/img/projects/RegistryEditor/srcset@0,25x.jpg
accent_color: '#cfdb5c'
accent_image:
  background: 'linear-gradient(to bottom,#cfdb5c 0%,#cfdb5c 30%,#d5e06d 50%,#d0d87d 70%,#dee595 100%)'
  overlay:    false

caption: Windows Registry Editor(regedit) with advanced search features.

description: >
  Windows Registry Editor(regedit) with advanced search features.

links:
  - title: Download
    url: https://github.com/giladreich/RegistryEditor
---

# Registry Editor

Registry Editor is a small utility that will help you finding specific registries with a specific search criteria.<br>
All the results will be showen in one place and you can then delete the selected registries or right click on one of the results will open 
the official Windows regedit in the location of the selected registry. Note that when deleting registries, a backup will automatically be 
created on your desktop.


## Motivation

If you're here, I guess you already aware that the official build in registry editor(regedit) in Windows allows you to only `Find Next` 
and searching for `Keys`, `Values` and `Data`.<br>
That sometimes can be very time consuming when you look for specific registry, therefore writing a quick gui application to add all 
these results in one place wouldn't take much time so I can just let it scan through with more advanced filters and do with the results 
whatever comes in mind, i.e deleting found results, saving, jumping directly to the official registry location etc... <br>
I also have to give credit to my brother that asked for a simple script to scan through the registry on a specific criteria, which brought 
me to the idea of writing a gui application that will be much easier to maintain and extend if needed.

Another useful situation is when you install a new software and you want to look for any registers that the software installed on your 
machine, which can be easily tracked down by scanning the whole registry and finding their existing locations.

## Demo

![alt text][img_main]

![alt text][img_edit]

![alt text][img_network]

[img_main]: /assets/img/projects/RegistryEditor/pictures/mainwindow.png
[img_edit]: /assets/img/projects/RegistryEditor/pictures/findwindow.png
[img_network]: /assets/img/projects/RegistryEditor/pictures/networkwindow.png
