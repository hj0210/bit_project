# 뉴스 크롤링

## Day3

- List comprehension을 사용한 for문! 처음에는 사용하기 쉽지않으나 익숙해지자

  ```python
  links = [tag['href'] for tag in news_list]
  과 같은 의미의 for문
  
      # links = []
      # for tag in news_list:
      #     links.append(tag['href'])
  ```

- 오늘은 같은 페이지 내의 뉴스기사들의 url들을 배열 형태로 추출하는 소스를 만들었다. 해당 소스에 들어가서 먼저 만들어진 소스를 합치고 page를 알아낸다면 모든 기사를 추출할 수 있을거로 예상된다

```python
from selenium import webdriver
from bs4 import BeautifulSoup
import re
import time
import calendar
import platform
import pandas as pd
import csv


class ArticleParser(object):
    special_symbol = re.compile('[\{\}\[\]\/?,;:|\)*~`!^\-_+<>@\#$&▲▶◆◇◀■【】\\\=\(\'\"]')
    content_pattern = re.compile(
        '본문 내용|TV플레이어| 동영상 뉴스|flash 오류를 우회하기 위한 함수 추가function flash removeCallback|tt|앵커 멘트|xa0')

    @classmethod
    def clear_content(cls, text):
        # 기사 본문에서 필요없는 특수문자 및 본문 양식 등을 다 지움
        newline_symbol_removed_text = text.replace('\\n', '').replace('\\t', '').replace('\\r', '')
        special_symbol_removed_content = re.sub(cls.special_symbol, ' ', newline_symbol_removed_text)
        end_phrase_removed_content = re.sub(cls.content_pattern, '', special_symbol_removed_content)
        blank_removed_content = re.sub(' +', ' ', end_phrase_removed_content).lstrip()  # 공백 에러 삭제
        reversed_content = ''.join(reversed(blank_removed_content))  # 기사 내용을 reverse 한다.
        content = ''
        for i in range(0, len(blank_removed_content)):
            # reverse 된 기사 내용중, ".다"로 끝나는 경우 기사 내용이 끝난 것이기 때문에 기사 내용이 끝난 후의 광고, 기자 등의 정보는 다 지움
            if reversed_content[i:i + 2] == '.다':
                content = ''.join(reversed(reversed_content[i:]))
                break
        return content

    @classmethod
    def clear_headline(cls, text):
        # 기사 제목에서 필요없는 특수문자들을 지움
        newline_symbol_removed_text = text.replace('\\n', '').replace('\\t', '').replace('\\r', '')
        special_symbol_removed_headline = re.sub(cls.special_symbol, '', newline_symbol_removed_text)
        return special_symbol_removed_headline

    @classmethod
    def find_news_totalpage(cls, url):
        # 당일 기사 목록 전체를 알아냄
        try:
            totlapage_url = url
            driver = webdriver.Chrome('D:\\git\\source\\chromedriver\\chromedriver.exe')
            driver.implicitly_wait(3)
            result = driver.get(totlapage_url)
            result
            time.sleep(1)
            html = driver.page_source
            document_content = BeautifulSoup(html, 'html.parser')
            headline_tag = document_content.find('div', {'class': 'paging'}).find('strong')
            regex = re.compile(r'<strong>(?P<num>\d+)')
            match = regex.findall(str(headline_tag))
            return int(match[0])
        except Exception:
            return 0


class ArticleCrawler(object):
    driver = webdriver.Chrome('D:\\git\\source\\chromedriver\\chromedriver.exe')

    def __init__(self):
        self.sid1 = {'정치': 100, '경제': 101, '사회': 102, '생활/문화': 103, '세계': 104, 'IT/과학': 105}
        self.sid2 = {'청와대': 264, '국회/정당': 265, '북한': 268, '행정': 266, '국방/외교': 267, '정치일반': 269,
                     '금융': 259, '증권': 258, '산업/재계': 261, '중기/벤처': 771, '부동산': 260, '글로벌경제': 262, '생활경제': 310,
                     '경제일반': 263,
                     '사건사고': 249, '교육': 250, '노동': 251, '언론': 254, '환경': 252, '인권/복지': '59b', '식품/의료': 255, '지역': 256,
                     '인물': 276, '사회일반': 257,
                     '건강정보': 241, '자동차/시승기': 239, '도로/교통': 240, '여행/레저': 237, '음식/맛집': 238, '패션/뷰티': 376, '공연/전시': 242,
                     '책': 243, '종교': 244, '날씨': 248, '생활문화일반': 245,
                     '아시아/호주': 231, '미국/중남미': 232, '유럽': 233, '중동/아프리카': 234, '세계일반': 322,
                     '모바일': 731, '인터넷/SNS': 226, '통신/뉴미디어': 227, 'IT일반': 230, '보안/해킹': 732, '컴퓨터': 283, '게임/리뷰': 229,
                     '과학일반': 228,
                     }
        # self.selected_sid1 = []
        # self.selected_sid2 = []
        # self.date = {'start_year': 0, 'start_month': 0, 'end_year': 0, 'end_month': 0}
        # self.user_operating_system = str(platform.system())

    # def set_sid1(self, *args):
    #     return self.sid1.get(args)
    #
    #
    # def set_sid2(self, *args):
    #     return self.sid2.get(args)

    def crawling(self, sid1, sid2):
        URL = 'https://news.naver.com/main/list.nhn?mode=LS2D&mid=shm&sid1=' + str(self.sid1.get(sid1)) + '&sid2=' + str(self.sid2.get(sid2))
        self.driver.implicitly_wait(3)
        self.driver.get(URL)
        html = self.driver.page_source
        soup = BeautifulSoup(html, 'html.parser')

        news_list =soup.select("div.list_body a")
        links = [tag['href'] for tag in news_list]


        self.driver.quit()
        print(links)

if __name__ == "__main__":
    Crawler = ArticleCrawler()
    Crawler.crawling('정치', '청와대')

```

