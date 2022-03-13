---
layout: post
title:  Scraping with Selenium and GitHub Actions
description: Unleashing the power webdriver-manager.
date: 2022-03-10
categories: journalism
---

## ⏳ TL;DR

*If you want to skip directly to the code, visit [this GitHub repository](https://github.com/aadittambe/selenium-scraper).*

## 🖼 Once upon a time in a land far far away ... 

For about half a year now, I have been a fan of writing web scrapers and automating them with [GitHub's powerful Actions feature](https://github.com/features/actions). The free GitHub feature — originally created for software engineers to build pipelines for testing, releasing and deploying software — is an [efficient way for journalists](https://palewi.re/docs/first-github-scraper/) to run Python web scrapers on a schedule, without having to set up a server.

One problem stumped me, though: "`requests` is cool, but how do I set up a `selenium` scraper with GitHub Actions, since it requires the installation of a WebDriver?"

While [`requests`](https://docs.python-requests.org/en/latest/) allows us to send HTTP requests to get data from websites, [`selenium`](https://selenium-python.readthedocs.io/) shines when we want to push buttons, fill input forms, scroll and take screenshots. 

As journalism educator Jonathan Soma explains in [this video](https://www.youtube.com/watch?v=mAwL_0N1W9E&t), Selenium is a browser automation software to control a web browser, such as Chrome or Firefox. However, in order for Selenium to pass commands to the web browser, it uses a WebDriver. So, the first step in building a Selenium web scraper is to set up a WebDriver, which can issue commands to the browser on behalf of Selenium.

There are two problems — headaches, rather — with this approach: 
1. As a web browser updates, the WebDriver for that browser also updates. As a result, every time your browser updates, you have to download a new WebDriver such that the versions are consistent. 
1. It's tricky to run this process on GitHub Actions, since this process happens on a virtual machine. There are potential [solutions](https://www.blazemeter.com/blog/automated-testing-selenium-github-actions) to circumvent this problem, but for running a simple web scraper, they can be overkill.

`webdriver-manager` automatically manages your WebDriver to make it match your browser version. It's slick because you don't have to download anything from the internet, and you can run the script with GitHub Actions.

An example: for a side project, I am scraping data for [journalists killed](https://cpj.org/data/killed/?status=Killed&motiveConfirmed%5B%5D=Confirmed&type%5B%5D=Journalist&start_year=1992&end_year=2022&group_by=year) on the line of duty every day. It's collected by the Committee to Protect Journalists, and lives in a tabular format on multiple pages. The URL does not change when I switch between pages, and, therefore, `requests` alone wouldn't do the trick. 

## 💪 Building the script & automating the process

Here's how I went about building the scraper using Python, `Selenium`, `webdriver-manager` and GitHub Actions:

### Step 1: Import libraries 
In the Python script, I install and imported all the libraries I was going to need. You may have to use `pip` to [install](https://pypi.org/project/webdriver-manager/) `webdriver-manager`.

```python
import pandas as pd
from bs4 import BeautifulSoup
import requests
import selenium
import csv
import time
```

### Step 2: Import webdriver-manager

Here, I have used a WebDriver for Chrome.
```python
from selenium import webdriver
from selenium.webdriver.chrome.options import Options
from webdriver_manager.chrome import ChromeDriverManager
```

### Step 3: Set up options for the WebDriver

Headless means that the WebDriver will not fire up a GUI. If you're testing your code, you may want to comment it out.
```
chrome_options = Options()
chrome_options.add_argument("--headless")
```

### Step 4: Initialize the WebDriver and point it to a website

In this step, `webdriver-manager` will install the ChromeDriver. And options will be set to the options we set up in Step 3. If you turned off `headless`, you will see a browser instance firing up in this step.
```python
driver = webdriver.Chrome(
        ChromeDriverManager().install(),
        options=chrome_options
        )
driver.get("https://cpj.org/data/killed")
```

### Step 5: Complete writing the scraper

The usual stuff ...

```python
list_of_rows = []
counter = 0
while counter < 100:
    html = driver.page_source
    soup = BeautifulSoup(html, 'html.parser')
    table = soup.find("table", {"table js-report-builder-table"})
    for row in table.find_all('tr'):
        list_of_cells = []
        for cell in row.find_all('td'):
            if cell.find('a'):
                list_of_cells.append(cell.find('a')['href'])
            text = cell.text.strip()
            list_of_cells.append(text)
        list_of_rows.append(list_of_cells)
    counter = counter + 1
    next_button = driver.find_element_by_xpath('/html/body/div[1]/div/div/div[2]/div/div[1]/div/nav/ul/li[8]/a')
    next_button.click()
    time.sleep(1)
data = pd.DataFrame(list_of_rows, columns=["link","name", "organization", "date", "location","killed","type_of_death", ""]).dropna()
data.to_csv("data.csv",index=False)
```

### Step 6: Safe the file, and automate the scraper with GitHub Actions

I have named this script file `scraper.py` and added it a [GitHub repository](https://github.com/aadittambe/selenium-scraper). This is what my Actions workflow looks like. The YAML file is stored in the `.github/workflows` directory, as `main.yml` — the Action is scheduled to run every day at 8 a.m. 

```yaml
name: Scrape

on:
  push:
  workflow_dispatch:
  schedule:
    - cron: "0 8 * * *" # 8 a.m. every day UTC

jobs:
  scrape:
    runs-on: ubuntu-latest
    steps:
    # Step 1: Prepare the environment
    - name: Check out this repo
      uses: actions/checkout@v2
      with:
        fetch-depth: 0
    # Step 2: Install requirements, so Python script can run
    - name: Install requirements
      run: python -m pip install pandas selenium requests bs4 webdriver-manager
    # Step 3: Run the Python script    
    - name: Run scraper
      run: python scrape.py     
    # Step 4: Commit and push
    - name: Commit and push
      run: |-
        git pull
        git config user.name "Automated"
        git config user.email "actions@users.noreply.github.com"
        git add -A
        timestamp=$(date -u)
        git commit -m "Latest data: ${timestamp}" || exit 0
        git push
```

If you're looking to further learn about GitHub Actions, check out [this](https://palewi.re/docs/first-github-scraper/index.html) lesson.

## 🧐 Moral of the story

In conclusion, you may never have to go back to downloading a WebDriver, and re-downloading it every time your browser updates, if you use this `webdriver-manager`.

Happy web scraping & automating! 🤖

*I hope this is helpful. If I've gotten something wrong — or if you think this method sucks — please do [let me know](mailto:aadit.tambe@gmail.com).*

