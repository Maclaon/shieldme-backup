---
title: Ubuntu上使用Xvfb来进行无界面化浏览器测试
date: 2017-03-18 10:00:00
tags: 自动化测试、xvfb、selenium
author: maclaon
comments: false
---

# 安装步骤
## 安装chrome
+ `sudo wget http://www.linuxidc.com/files/repo/google-chrome.list -P /etc/apt/sources.list.d/`
+ `wget -q -O - https://dl.google.com/linux/linux_signing_key.pub  | sudo apt-key add -`
+ `sudo apt-get update`
+ `sudo apt-get install google-chrome-stable`

## 安装xvfb
+ `sudo apt-get install google-chrome-stable`

## 安装ChromeDriver
+ `sudo apt-get install unzip`
+ `wget -N http://chromedriver.storage.googleapis.com/2.XX/chromedriver_linux64.zip`
+ `unzip chromedriver_linux64.zip`
+ `sudo mv -f chromedriver /usr/local/share/chromedriver`
+ `sudo ln -s /usr/local/share/chromedriver /usr/local/bin/chromedriver`
+ `sudo ln -s /usr/local/share/chromedriver /usr/bin/chromedriver`

# 使用方式

<!--more-->

## Python
### 安装
+ `sudo apt-get install python-pip`
+ `pip install pyvirtualdisplay selenium`

### 脚本
```python
from pyvirtualdisplay import Display
from selenium import webdriver

display = Display(visible=0, size=(800, 600))
display.start()
chrome_options = webdriver.ChromeOptions()
chrome_options.add_argument('--no-sandbox')
driver = webdriver.Chrome('/usr/local/bin/chromedriver', chrome_options=chrome_options)

driver.get('http://www.baidu.com')
print driver.title
```

# 参考文献
+ [Installing Selenium and ChromeDriver on Ubuntu](https://christopher.su/2015/selenium-chromedriver-ubuntu/)
