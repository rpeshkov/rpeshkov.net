+++
date = "2016-10-16T22:42:54+03:00"
title = "Cyrillic symbols on english Windows 10"
categories = ["Tips"]
+++

I use english version of Windows 10 but sometimes I have to deal with files that have cyrillic names or with cyrillic
commit messages in TortoiseHg.

I've found that my favorite image viewer [XnView](http://xnview.com) doesn't work properly with cyrillic paths when
installed on english Windows:

![XnView](/images/cyrillic-symbols-on-english-windows/xnview.png)

TortoiseHg Workbench have some issues too:

![TortoiseHg Workbench](/images/cyrillic-symbols-on-english-windows/question-marks.png)

That particular issues mean that XnView and TortoiseHg Workbench doesn't support Unicode. However, there's a fix for
this by applying some settings in Windows. You need to tell Windows which language to use for non-Unicode applications.

1) Navigate to Control Panel, either by Start Menu or by running `control` command.

![Start Menu](/images/cyrillic-symbols-on-english-windows/start-menu.png)

2) Open "Language" item in Control Panel and select "Change date, time, or number formats".

![Language](/images/cyrillic-symbols-on-english-windows/control-panel-language.png)

3) Navigate to "Administrative" tab and push "Change system locale..." button.

![Administrative tab](/images/cyrillic-symbols-on-english-windows/language-administrative.png)

4) Select "Russian (Russia)" option from dropdown and save all the windows that you've opened.

5) Restart your computer.

When your computer will restart, XnView will be able to work with cyrillic paths and TortoiseHg will display commit
messages correctly.

