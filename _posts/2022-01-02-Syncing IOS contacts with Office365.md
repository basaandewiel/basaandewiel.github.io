---
layout: post
title: Syncing IOS contact with Office365 
---
## The problem
I am using Windows laptop with Office365 and recently switched from an Android phone to Apple IOS.
Question was how to handle my contacts.
Should I put everything in iCloud and install the Outlook iCloud plugin, ro not?

I ended up in all my contacts in both Office365 (Outlook) AND IOS contacts (iCloud). And if I added a contact on my phone (for instance when you receive a call from an until then unknown number), that contact appeared only in IOS contacts, and not in Office365.

## The solution
I like to have a simple architecture
* I want all my contacts in Office365; Office365 is leading\
* I want that new contacts also arrive in Office365
* As a consequence I have no contacts at all in IOS

This can be arranged as follows:
* laptop: if you have installed icloud for Windows then turn off syncing of contact/agenda to icloud
* iphone: settings->icloud; switch slide at 'contacts' to off (this makes new contacts go to Office365)
* iphone: contacts->groups: let only shwo the Exchange contacts (if you have more than one contacts map in Exchange, select the correct one(s))
* iphone: now you can delete all contacts from iCloud (if you have these also in Office365)
