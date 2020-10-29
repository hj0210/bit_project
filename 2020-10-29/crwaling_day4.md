# 뉴스 크롤링

## Day4

- 문제점1)
  - 뉴스를 볼 수 있는 경로가 사진, 텍스트 2가지경로로 중복되는 리스트를 제거 시키는 set 함수 사용
    시간별로 나올 수 있게 해야하는데 set은 순서가 상관없이 됨. 그러니까 입력된 순서로 나올 수 있게 해결방안 생각해야함

```python
news_list =soup.select("div.list_body a")
links = [tag['href'] for tag in news_list]
links = set(links)
```

- 해결방안1)
  - 사진이 없는 뉴스도 있기 때문에 글자로만 접근한다고 생각한다.
  - 사진 경로로 들어가는 태그가 아닌것으로 접근하겠다

```python
news_list = soup.select("div.list_body dt[class!='photo'] > a")
links = [tag['href'] for tag in news_list]
```





- 현재 상태

```python
from selenium import webdriver
from bs4 import BeautifulSoup
import re
import time
from time import sleep
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

    def url_maker(self, url, start_year, start_month, end_year, end_month):
        made_url_list = []
        self.driver.implicitly_wait(3)
        self.driver.get(url)

        for year in range(start_year, end_year + 1):
            if start_year == end_year:
                year_startmonth = start_month
                year_endmonth = end_month
            else:
                if year == start_year:
                    year_startmonth = start_month
                    year_endmonth = 12
                elif year == end_year:
                    year_startmonth = 1
                    year_endmonth = end_month
                else:
                    year_startmonth = 1
                    year_endmonth = 12

            for month in range(year_startmonth, year_endmonth + 1):
                for month_day in range(1, calendar.monthrange(year, month)[1] + 1):
                    if len(str(month)) == 1:
                        month = "0" + str(month)
                    if len(str(month_day)) == 1:
                        month_day = "0" + str(month_day)

                    # 날짜별로 Page Url 생성
                    page_url = url + '&' + '&date=' + str(year) + str(month) + str(month_day)

                    # 네이버 페이지 구조를 이용해서 page=999으로 지정해 해당 카테고리 뉴스의 총 페이지 수를 알아냄
                    # page=999을 입력할 경우 페이지가 존재하지 않기 때문에 page=totalpage로 이동 됨 (Redirect)
                    lastpage_url = page_url + "&page=999"
                    self.driver.get(lastpage_url)
                    html = self.driver.page_source
                    document_content = BeautifulSoup(html, 'html.parser')

                    lastpage_tag = document_content.select(".paging strong")[0]
                    # lastpage_tag = document_content.find('div', {'id': 'paging'})

                    for page in range(1, int(lastpage_tag.text) + 1):
                        made_url_list.append(page_url + "&page=" + str(page))
        return made_url_list

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

    def crawling(self, sid1, sid2):
        URL = 'https://news.naver.com/main/list.nhn?mode=LS2D&mid=shm&sid1=' + str(self.sid1.get(sid1)) + '&sid2=' + str(self.sid2.get(sid2))

        self.driver.implicitly_wait(3)
        self.driver.get(URL)
        html = self.driver.page_source
        soup = BeautifulSoup(html, 'html.parser')

        news_list = soup.select("div.list_body dt[class!='photo'] > a")
        links = [tag['href'] for tag in news_list]

        # csv 파일로 담을 수 있게 배열을 하나 더 생성한다.
        result = []
        for content_url in links:
            sleep(0.01)

            self.driver.get(content_url)

            try:
                html_content =self.driver.page_source
                soup_content =BeautifulSoup(html_content, 'html.parser')

            except:
                continue

            try:
                title_tag = soup_content.find('h3', {'class': 'tts_head'})
                title = ''
                title = title + ArticleParser.clear_headline(str(title_tag.text))

                write_date_tag = soup_content.find('span', {'class': 't11'})

                main_text_tag = soup_content.find('div', {'class': '_article_body_contents'})
                # 불필요한 태그 제거
                for main_text in soup_content.find_all('script'):
                    main_text.extract()
                main_text = ''
                main_text = main_text + ArticleParser.clear_content(str(main_text_tag.text))

                for info in soup_content:
                        temp = []
                        write_date = write_date_tag.text
                        temp.append(write_date)
                        temp.append(sid1)
                        temp.append(sid2)
                        temp.append(title)
                        temp.append(main_text)
                        temp.append(content_url)
            except:
                continue


            result.append(temp)
        print(result)
        return result


if __name__ == "__main__":
    Crawler = ArticleCrawler()
    Crawler.crawling('정치', '청와대')
    # Crawler.시작날짜(20200101, 20201231)

    Crawler.driver.quit()
```

- 현재 상태에 대한 결과

```shell
[['2020.10.29. 오후 5:33', '정치', '청와대', '文대통령 청문회 기피현상...반드시 제도 개선돼야', '좋은\xa0인재\xa0모시기\xa0정말\xa0쉽지\xa0않아 다음\xa0정부에서라도\xa0풍토\xa0벗어나야 서울 뉴스1 유승관 기자 문재인 대통령이 28일 서울 여의도 국회 의장접견실에서 2021년도 예산안 시정연설에 앞서 박병석 국회의장 및 여야지도부 등과 환담하고 있다. 왼쪽부터 정세균 국무총리 유남석 헌법재판소장 김명수 대법원장 문 대통령 박병석 국회의장 더불어민주당 이낙연 대표 정의당 김종철 대표. 김종인 국민의힘 비상대책위원장은 라임·옵티머스 사태 특검 수용을 요구하며 이날 사전 환담에 불참했다. 2020.10.28 뉴스1 사진 뉴스1화상 파이낸셜뉴스 문재인 대통령이 현행 인사청문회 제도 의 개선 필요성을 강하게 역설했다. 지난 28일 국회에서 열린 2012년도 예산안 시정연설 전 국회의장 및 여야 지도부와의 비공개 사전 환담 자리에서다. 문 대통령은 좋은 인재를 모시기가 정말 쉽지 않다 며 청문회 기피현상이 실제로 있다. 본인이 뜻이 있어도 가족이 반대해서 좋은 분들을 모시지 못한 경우도 있다 고 아쉬움을 토로했다고 강민석 청와대 대변인은 29일 브리핑을 통해 밝혔다. 인사청문회가 공직후보자에 대한 신상털기식이 아닌 깊이 있는 자질 검증이라는 본래 취지를 살려야 한다는 것이 문 대통령의 생각이다. 특히 박병석 국회의장이 국회에서도 후보자의 도덕성 검증은 비공개로 하고 정책과 자질 검증은 공개로 하는 방향으로 청문회 제도를 고치려고 하고 있다 고 소개하자 문 대통령은 그 부분은 반드시 개선됐으면 좋겠다 고 희망을 피력했다. 문 대통령은 또 우리 정부는 종전대로 하더라도 다음 정부는 벗어나야 한다 며 다음 정부에서라도 기존의 인사청문회 풍토와 문화 탈피를 강조했다. 그러면서 다음 정부에서는 반드시 길이 열렸으면 한다 고 재차 강조했다고 강 대변인은 전했다. 현재 국회에는 더불어민주당 홍영표 정성호 의원이 대표발의한 인사청문회법 개정안이 발의되어 있다. 홍 의원은 최근 들어 인사청문회가 공직후보자에 대한 과도한 인신공격 또는 신상털기에 치중한 나머지 공직후보자의 자질을 검증하기 위한 본래의 기능을 상실하고 있다는 지적이 있다 며 개정안 발의 배경을 설명했다. 다만 개정안에 대한 국회 논의는 속도를 내지 못하고 있다. 문 대통령의 인사청문회 제도 언급에 대해 일각에서는 개각을 염두에 둔 것 아니냐 는 관측이 나왔지만 청와대는 말을 아꼈다. 청와대 핵심관계자는 인사 문제에 대해서는 미리 언급하지 않겠 다 며 인사청문회 제도와 관련한 계기 라고만 했다. 한편 이날 환담에서 유명희 통상교섭본부장이 세계무역기구 WTO 사무총장 선거 결선에 진출한 것과 관련 김영춘 국회 사무총장은 승패에 상관없이 문 대통령이 후보 연좌제를 깬 것 이라고 의미를 부여했다. 유 본부장의 남편은 정태옥 전 국민의힘 의원이다. 이에 문 대통령은 부부는 각각의 인격체 아닌가. 각자 독립적으로 자유로운 활동을 하는 것 이라고 말했다.', 'https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=100&sid2=264&oid=014&aid=0004519728'], ... ['2020.10.29. 오후 3:27', '정치', '청와대', '文 청문회 기피 현상…좋은 인재 모시기 쉽지 않다종합', '시정연설 전 사전 환담… 인청법 개정 에 文 반드시 개선 우리 정부는 종전대로 해도 다음 정부서 길 열렸으면 개각 어려움 반영 질문엔 靑 제도 관련…인사 언급 않겠다 서울 뉴시스 박영태 기자 문재인 대통령이 지난 28일 서울 여의도 국회 의장 접견실에서 2021년도 예산안 시정연설을 하기 전 박병석 국회의장과 환담을 하고 있다. 2020.10.28. since1999 newsis.com 서울 뉴시스 안채원 홍지은 기자 문재인 대통령은 지난 28일 2021년도 예산안 시정연설 전 비공개 환담에서 인사청문회도 가급적 본인을 검증하는 과정이 되어야 하지 않겠나 라며 망신주기식 인사청문회 풍토가 달라져야 한다는 뜻을 내비쳤다. 강민석 청와대 대변인은 29일 춘추관 브리핑을 갖고 전날 문 대통령과 박병석 국회의장 간 대화를 전하며 이같이 밝혔다.전날 열린 사전환담에서 박 의장은 국회에서도 후보자의 도덕성 검증은 비공개로 하고 정책과 자질 검증은 공개하는 방향으로 청문회 과정을 고치려 하고 있다 며 현재 국회에는 청문회법 개정안까지 발의돼있는 상태지만 현재 논의 속도가 나지 않고 있다 고 말했다.홍영표 더불어민주당 의원은 지난 6월 장관 후보자 등에 대한 국회 인사청문회의 도덕성 검증 부문을 비공개로 하는 내용을 골자로 하는 인사청문회법 개정안을 발의했다. 그러자 문 대통령은 그 부분은 반드시 개선됐으면 한다 며 우리 정부는 종전대로 하더라도 다음 정부는 벗어나야 한다 고 말했다. 강 대변인은 작금의 인사청문회 풍토 문화에서 다음 정부는 벗어나야 한다는 취지 라고 설명했다.문 대통령은 그러면서 좋은 인재를 모시기가 정말 쉽지 않다. 청문회 기피 현상이 실제 있다 며 본인이 뜻이 있어도 가족이 반대해서 좋은 분을 모시지 못한 경우도 있다. 다음 정부에서는 반드시 길이 열렸으면 한다 고 재차 강조했다고 강 대변인은 전했다.청와대 핵심 관계자는 인사청문회가 공직사회 도덕성을 한층 끌어올리는 순기능이 있는 건 사실 이라면서도 현재 청문회 기피가 심각한 수준이라면 나라를 위해서 좋지 않다. 지금 정부도 다음 정부도 마찬가지 라고 했다.이어 후보자 본인보다 주변에 대한 얘기들이 많고 심지어 며느리의 성적증명서까지 요구하는 상황 이라며 문 대통령의 발언은 우리 정부에서 제도 개선이 이뤄지지 않는다면 다음 정부라도 반드시 개선을 해야한다는 진정성을 담은 발언 이라고 전했다.문 대통령이 개각을 준비하는 데 어려움이 있어서 관련 발언이 나온 것이냐는 질문에는 인사청문회 제도와 관련한 이야기 라며 개각을 하는지 안 하는지도 공개하지 않았다. 인사 문제는 언급하지 않겠다 고 답했다.', 'https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=100&sid2=264&oid=003&aid=0010156338']]
```



- 추후 일정
  - 한 페이지에 있는 뉴스기사를 크롤링하는 것은 완성 됐다.
  - 데이터 프레임으로 만들기 위한 배열의 형태도 완성 됐다.
  - 원하는 날짜에 대한 범위, page 수를 인식하는 함수를 완성하면  뉴스 크롤링 끝