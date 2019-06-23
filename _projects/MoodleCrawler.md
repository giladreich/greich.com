---
layout: project
title: MoodleCrawler
date: 16 Oct 2018
screenshot:
  src: /assets/img/projects/MoodleCrawler/srcset@0,25x.jpg
  srcset:
    1920w: /assets/img/projects/MoodleCrawler/srcset@1x.jpg
    960w: /assets/img/projects/MoodleCrawler/srcset@0,5x.jpg
    480w: /assets/img/projects/MoodleCrawler/srcset@0,25x.jpg
accent_color: '#0b9885'
accent_image:
  background: 'linear-gradient(to bottom,#0b9885 0%,#0b9885 30%,#25bca8 50%,#48cebc 70%,#73e2d4 100%)'
  overlay:    false

caption: Web crawler for downloading all resources from Moodle servers.

description: >
  Web crawler for downloading all resources from Moodle servers.

links:
  - title: Download
    url: https://github.com/giladreich/MoodleCrawler
---

# Moodle Server Crawler / Downloader

Your school is temporary shutting down the moodle server for whatever reason? No worries! MoodleCrawler for the rescue ;)
This is a simple single threaded moodle crawler GUI application that will help you downloading all resources that are attached by your teachers.
It will save all resources in the directory structure as shows in the moodle server and also download some of the pages as HTML format.

Written in C# using Windows Forms.

---
## How-To-Use

The only thing you need to do is to the set the UrlMoodleRoot settings of your moodle server in your configuration file as follows:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
    <configSections>
        <sectionGroup name="userSettings" type="System.Configuration.UserSettingsGroup, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" >
            <section name="Moodle.WebCrawler.Properties.Settings" type="System.Configuration.ClientSettingsSection, System, Version=4.0.0.0, Culture=neutral, PublicKeyToken=b77a5c561934e089" allowExeDefinition="MachineToLocalUser" requirePermission="false" />
        </sectionGroup>
    </configSections>
    <startup> 
        <supportedRuntime version="v4.0" sku=".NETFramework,Version=v4.7"/>
    </startup>
    <userSettings>
        <Moodle.WebCrawler.Properties.Settings>
            <setting name="UrlMoodleRoot" serializeAs="String">
                <value>https://moodle.schoolname.com</value>
            </setting>
        </Moodle.WebCrawler.Properties.Settings>
    </userSettings>
</configuration>
```

NOTE: If you are building from source, right click on the project `Moodle.WebCrawler` `->` `Properties` `->` `Settings` and then change the value to your school url.

I would also highly recommend running in debug mode using VS since it's a single threaded application and I didn't take the time
to make the GUI to display the requests activity, so you'll see all activity in the `Output window` of VS. You could also compile the project as console application so while you use the GUI, the requests will be outputted in the console window.
Other optional tools can be used during crawling like `Wireshark` or `Fiddler` to see the requests being sent in the background.

---
## Motivation

My school is shutting down the moodle server for some time 10 days before our final exam, therefore I though it wouldn't take much time and effort 
to crawl all that data on the server for an offline version.
Also, sometimes you don't have internet access when you're on the ways, so it's always good to have a backup.
