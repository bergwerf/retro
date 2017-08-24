---
layout:      post
title:       Google Drive OBJ viewer
date:        2013-06-01
categories:  projects
author:      Herman Bergwerf
---
Drive3D is a Google Drive app which can display OBJ files in your browser,
Drive3D is made using Three.js.

I wanted to make a Google Drive app. Currently, 3D files are not viewable in
Google Drive, so I decided to write an app that can display OBJ/MTL files
directly in your browser. I found an example which used Three.js to load and
render a locally stored OBJ file and combined it with the Google Drive API. I
also used the `dat.GUI` JavaScript library to add some extra controls. You can
set the background (clear) color, the light color and the light intensity. You
can even embed the OBJ file into your website by making the folder where the OBJ
file is located public, and pasting the embed code into your website!

Drive3D is available in the [Chrome Web Store](https://chrome.google.com/webstore/detail/drive3d/nhcndmbdddjkjghhhjnlgfbolioepgkn) or you can [Launch Drive3D](https://drive3d.hermanbergwerf.com) directly.
