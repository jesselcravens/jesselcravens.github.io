---
layout: post
title:  Multiple Server Pull and Push Files Application
date:   2024-08-13 17:56:32 -0500
categories: Python
---
# Multiple Server Pull and Push Files Application



This is a tool I created to help manage manual backups and upgrades of multiple Linux servers at once. I call the tool PulliGUI based on its first function which was to only pull backup files. 



#### Capabilities:

- Can currently support up to 60 servers at once
- Flexible filenames and paths for files to pull from the servers
- Ability to push files to servers
- Ability to open the IP addresses selected in a web browser.  This is nice for servers with Web based user interfaces.  
- Flexible user name and passwords so that nothing is hard coded for security. 



#### Screenshot:

![](/assets/PulliGUI.PNG)

#### Code for Review

Full code for the interested in another github project: https://github.com/jesselcravens/pulliGUI/blob/main/app.py



#### Interesting facts about how I coded this: 

- There is a limit currently of 60 servers
  - This is accomplished because the application creates 60 boxes/spots when it starts. 
  - Then the boxes are populated based on the server_list.csv file
  - Then the empty boxes are removed to cleanup the display to only show how many boxes/spots have server details. 
- When opening a webpage, the code will look for a Windows Chrome installation first.
  - If no Chrome installation is found, it will attempt to use Firefox next
  - If no Firefox, then it will default to the default web browser. 
  - My reasoning was: I want to use something other than Microsoft EDGE as a default

