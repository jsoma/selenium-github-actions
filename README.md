# Make Selenium work on Github Actions 

Scraping with BeautifulSoup on GitHub Actions is easy-peasy. But what about Selenium?? After you jump through some hoops it's *just as simple.*

## How to use this

I think you can just fork this repository and get to work on `scraper.py`.

Otherwise, just steal `scraper.py` and `.github/workflows/scrape.yml` and you'll be good to go.

### Saving CSV files

If you want every scrape to update a CSV or something like this, you'll need to edit your `scrape.yml` to commit to the repo after the scraper is done running.

Take a look at the bottom of [autoscraper-history](https://github.com/jsoma/autoscraper-history/blob/main/.github/workflows/scrape.yml) to see how to do that. It should be one more step, something like this:

```yaml
      - name: Commit and push if content changed
        run: |-
          git config user.name "Automated"
          git config user.email "actions@users.noreply.github.com"
          git add -A
          timestamp=$(date -u)
          git commit -m "Latest data: ${timestamp}" || exit 0
          git push
```

I'm pretty sure I lifted that code 100% from [Simmon Willison](https://simonwillison.net/2020/Oct/9/git-scraping/).

### Scheduling

Right now the yaml is set to only scrape when you click the "Run workflow" button in the Actions tab, but you can add something like

```yaml
  schedule:
    - cron: '0 * * * *'
```

To make it run the first minute of every hour.

## How this all works

To make Selenium + GitHub Actions work, this repo does a few magic things. In a normal world, you start Chrome like this:

```python
from selenium import webdriver

driver = webdriver.Chrome()
```

But we.... do things a little differently. Let me walk you through the changes!

### [webdriver-manager](https://pypi.org/project/webdriver-manager/) to manage the webdriver

You can add a special action to [set up Chromedriver](https://github.com/marketplace/actions/setup-chromedriver) but I feel it's honestly easier to use webdriver-manager. It magically picks out the right version of chromedriver for you.

```python
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager

driver = webdriver.Chrome(ChromeDriverManager().install())
```

It works on your local machine, too!

### Chromium, not Chrome

When you're running GitHub Actions, it's probably on a nice little Ubuntu Linux machine. In those situations, you install software using `apt`. Since you can't install Chrome with `apt`, you'll install [Chromium](https://www.chromium.org/) instead, the open-source version of Chrome. Works the same, just opens a little differently.

As a result, our chromedriver install code changed a little more:

```python
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from webdriver_manager.utils import ChromeType

driver_path = ChromeDriverManager(chrome_type=ChromeType.CHROMIUM).install()
driver = webdriver.Chrome(driver_path)
```

This also means we need to add a line that does `apt-get install -y chromium-browser` to our [`scrape.yml`](.github/workflows/scrape.yml))

### Headless Chromium

Since we don't have a monitor plugged into GitHub Actions, we can't actually see what's going on in the browser. In the olden days you had to construct some odd technical fake screen, but these days you just run in headless mode!

```python
from selenium import webdriver 
from selenium.webdriver.chrome.options import Options

chrome_options = Options()
chrome_options.add_argument("--headless")
driver = webdriver.Chrome(options=chrome_options)
```

### Other options and more management of things

To combine the webdriver-manager driver path and the headless Chrome option, there are a lot of hoops to jump through. During the research process a lot of extra chrome options poked up, so I thought hey, let's just add all those, too.

```python
from selenium import webdriver
from webdriver_manager.chrome import ChromeDriverManager
from webdriver_manager.utils import ChromeType
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.chrome.service import Service

chrome_service = Service(ChromeDriverManager(chrome_type=ChromeType.CHROMIUM).install())

chrome_options = Options()
options = [
    "--headless",
    "--disable-gpu",
    "--window-size=1920,1200",
    "--ignore-certificate-errors",
    "--disable-extensions",
    "--no-sandbox",
    "--disable-dev-shm-usage"
]
for option in options:
    chrome_options.add_argument(option)

driver = webdriver.Chrome(service=chrome_service, options=chrome_options)
```

The biggest change here beyond all those options is the `Service` thing. Apparently just giving it the path to `chromdriver` isn't good enough? Who knows, I just do what works.