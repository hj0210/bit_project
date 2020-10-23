# 뉴스 크롤링

## Day1

- Selenium, BeautifulSoup 모듈 사용
- 문제 1: 데이터 본문을 불러오는 과정에서 본문내용과 무관한 tag들의 내용까지 포함되어 버렸다.
  - 데이터 정제 작업이 필요함
- 문제 2: 반복문이 여러개 들어간 문자열들을 한 컬럼에 넣어버려야하는데 그렇게되면 문자열을 하나하나 쪼개버려서 한컬럼에 들어가지 못한다. 이걸 한번에 묶어버리는 작업이 필요한데 해결하지 못했다.

```python
from selenium import webdriver
from bs4 import BeautifulSoup
import re
import pandas as pd
import csv


URL = 'https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=101&sid2=259&oid=421&aid=0004945097'
driver = webdriver.Chrome('D:\\git\\source\\chromedriver\\chromedriver.exe')
driver.implicitly_wait(3)
result = driver.get(URL)
html = driver.page_source
soup = BeautifulSoup(html, 'html.parser')
TAG_RE = re.compile(r'<[^>]+>')

def text():
    def remove_tags(text):
        return TAG_RE.sub('', text)
    results = list()
    for s in main_text_tag.select("script"):
        results.append(s.extract())
    for main_text in main_text_tag:
        results.append(remove_tags(str(main_text)))

    return results

results = []

title_tag = soup.find('h3', {'class': 'tts_head'})
write_date_tag = soup.find('span', {'class': 't11'})
main_text_tag = soup.find('div', {'class': '_article_body_contents'})




for info in soup:
    temp = []
    title = title_tag.text
    write_date = write_date_tag.text
    temp.append(title)
    temp.append(write_date)
    temp.append(text())
    results.append(temp)



data = pd.DataFrame(results)
data.columns = ['title', 'datetime', 'information']
data
```

![image-20201023210750252](C:\Users\USER\AppData\Roaming\Typora\typora-user-images\image-20201023210750252.png)