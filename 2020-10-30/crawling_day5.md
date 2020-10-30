# 뉴스크롤링

## Day5

- 카테고리1, 카테고리2, 시작날짜년도, 시작날짜월, 종료날짜년도, 종료날짜월 을 실행시키게 되면 데이터 프레임형태로 저장이 성공햇다
- 실행시키면 컬럼이름이 제대로 나오지 않는데 그부분만 수정하면 될것같다.
- csv 파일로 저장하게되면 한글이 깨지게되는데 인코딩 (UTF-8) 로 실행하면 한글이 깨지지 않고 나온다.

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

    def crawling(self, sid1, sid2, start_year, start_month, end_year, end_month):
        URL = 'https://news.naver.com/main/list.nhn?mode=LS2D&mid=shm&sid1=' + str(self.sid1.get(sid1)) + '&sid2=' + str(self.sid2.get(sid2))

        made_url_list = []
        self.driver.implicitly_wait(3)
        self.driver.get(URL)

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
                # for month_day in range(1, 3): 작동 잘되는지 확인할때 쓰는 범위
                    if len(str(month)) == 1:
                        month = "0" + str(month)
                    if len(str(month_day)) == 1:
                        month_day = "0" + str(month_day)

                    # 날짜별로 Page Url 생성
                    page_url = URL + '&date=' + str(year) + str(month) + str(month_day)

                    # 네이버 페이지 구조를 이용해서 page=999으로 지정해 해당 카테고리 뉴스의 총 페이지 수를 알아냄
                    # page=999을 입력할 경우 페이지가 존재하지 않기 때문에 page=totalpage로 이동 됨 (Redirect)
                    lastpage_url = page_url + "&page=999"
                    self.driver.get(lastpage_url)
                    html = self.driver.page_source
                    document_content = BeautifulSoup(html, 'html.parser')

                    lastpage_tag = document_content.select(".paging strong")[0]

                    for page in range(1, int(lastpage_tag.text) + 1):
                        made_url_list.append(page_url + "&page=" + str(page))

        # csv 파일로 담을 수 있게 배열을 하나 더 생성한다.
        result = []

        for URLS in made_url_list:

            self.driver.implicitly_wait(3)
            self.driver.get(URLS)
            html = self.driver.page_source
            soup = BeautifulSoup(html, 'html.parser')

            news_list = soup.select("div.list_body dt[class!='photo'] > a")
            links = [tag['href'] for tag in news_list]


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
        # print(result)
        data = pd.DataFrame(result)
        # \xa0, \xa9 를 없애줌
        data_transpose = data.applymap(lambda x: x.replace('\xa0','').replace('\xa9',''))
        data_transpose.to_csv("D:\crawling_result\\test.csv", header=False, index=False)






if __name__ == "__main__":
    Crawler = ArticleCrawler()
    Crawler.crawling('정치', '청와대', 2020, 8, 2020, 8)

    # Crawler.시작날짜(20200101, 20201231)

    # Crawler.driver.quit()
```

```shell
2020.08.01. 오후 10:48,정치,청와대,대통령직속 4차산업혁명委 데이터 옴부즈만 활동 개시,공공데이터관련민간의견및애로사항청취 문재인 대통령이 18일 강원 춘천시 남산면 더존비즈온 강촌캠퍼스에서 열린 한국판 뉴딜 디지털경제 현장방문 에 참석해 직원들에게 자리를 권하고 있다. 뉴시스 사진 fnDB 파이낸셜뉴스 대통령직속 4차산업혁명위원회 위원장 윤성로 는 8월부터 데이터 옴부즈만 활동을 시작한다. 한국판 뉴딜의 양대 축 가운데 하나인 디지털 뉴딜 사업의 핵심인 데이터 댐의 실질적 이용 활성화를 위해 현장의 의견과 애로사항을 청취하는 것이다. 4차산업위는 데이터 3법 개인정보보호법 정보통신망법 신용정보법 개정안 통과를 계기로 데이터 경제 활성화를 위한 정부·민간의 소통 플랫폼으로서 데이터 옴부즈만 역할을 맡고 있다. 지난 6월에는 개인정보보호법 시행령 개정안에 대한 민간 의견을 수렴하여 시행령 주요 쟁점 및 검토 의견을 관계 부처에 전달한 바 있다. 이어 정부의 디지털 뉴딜 발표에 따라 대표사업인 데이터 댐 의 핵심 자원인 공공데이터의 활용성 제고 및 부가가치 창출을 극대화하기 위해 민간의 의견 및 애로사항을 청취한다는 방침이다. 정부가 적극적으로 공공데이터 정책을 펼치고 있지만 여전히 국민·기업 등의 요구에 부응한 고수요·고부가가치 데이터 개방 및 활용은 미흡한 상황이다. 공공데이터의 활용성과 가치창출을 극대화하기 위해서는 △기업 및 민간이 원하는 고부가가치 데이터 개방 △비즈니스 측면에서 바로 활용할 수 있는 다양한 형태의 비정형 원천데이터 오픈 API CSV 등 데이터 제공 △표준화를 통한 유통·거래·융합기반 마련 등 수요자 중심의 시각과 정책이 필요하다. 이에 4차위는 공공데이터를 활용 중인 기업 전문가 등에 대한 심층 면담 등을 통해 현장의 의견을 수렴하고 공공데이터 개방 관련 주요 이해당사자 및 전문가가 참여하는 연구반을 구성 민간 의견을 검토하고 개선 방향을 제언할 계획이다. 윤성로 위원장은 데이터 옴부즈만 활동을 통해 도출된 핵심과제는 관계 부처와의 긴밀한 협의를 통해 정책에 반영될 수 있도록 하겠다 며 대규모 데이터를 축적하는 디지털 댐에서 나아가 그 데이터가 필요한 적재적소에 활용될 수 있도록 4차위가 민관의 소통채널이 되어 디지털 뉴딜 성공의 밑거름 역할을 하겠다 고 강조했다.,https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=100&sid2=264&oid=014&aid=0004470466
2020.08.01. 오후 6:17,정치,청와대,UAE 한국형 원전 바라카 1호기 시험 운전,서울 연합뉴스 셰이크 무함마드 빈 라시드 알막툼 아랍에미리트 UAE 총리 부통령 겸 두바이 지도자는 1일 현지시간 아부다비 바라카 원자력 발전소 1호기를 가동하기 시작했다고 밝혔다. 바라카 원전사업은 한국형 차세대 원전 APR1400 4기 총발전용량 5천60㎿ 를 UAE 수도 아부다비에서 서쪽으로 270㎞ 떨어진 바라카 지역에 건설하는 프로젝트다. 한국전력 컨소시엄은 2009년 12월 이 사업을 수주해 2012년 7월 착공했다. 애초 2017년 상반기 안으로 1호기를 시험 운전할 계획이었지만 UAE 정부 측에서 안전 자국민 고급 운용 인력 양성 등을 이유로 운전 시기를 수차례 연기했다. 상업 가동은 이르면 올해 안으로 시작되는 것으로 알려졌다.,https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=100&sid2=264&oid=001&aid=0011785124
2020.08.01. 오후 6:17,정치,청와대,UAE 한국형 원전 바라카 1호기 시험 운전,서울 연합뉴스 셰이크 무함마드 빈 라시드 알막툼 아랍에미리트 UAE 총리 부통령 겸 두바이 지도자는 1일 현지시간 아부다비 바라카 원자력 발전소 1호기를 가동하기 시작했다고 밝혔다. 바라카 원전사업은 한국형 차세대 원전 APR1400 4기 총발전용량 5천60㎿ 를 UAE 수도 아부다비에서 서쪽으로 270㎞ 떨어진 바라카 지역에 건설하는 프로젝트다. 한국전력 컨소시엄은 2009년 12월 이 사업을 수주해 2012년 7월 착공했다. 애초 2017년 상반기 안으로 1호기를 시험 운전할 계획이었지만 UAE 정부 측에서 안전 자국민 고급 운용 인력 양성 등을 이유로 운전 시기를 수차례 연기했다. 상업 가동은 이르면 올해 안으로 시작되는 것으로 알려졌다.,https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=100&sid2=264&oid=001&aid=0011785123
2020.08.01. 오후 4:32,정치,청와대,정 총리 대전 집중호우 피해 현장 점검…재발 방지책 세워야,침수 피해 발생한 대전 코스모스아파트 현장 점검 서울 뉴시스 김명원 기자 정세균 국무총리가 1일 오전 대전 서구 코스모스 아파트를 방문해 수해 피해 복구 현장을 돌아보고 있다. 사진 총리실 제공 2020.08.01. photo newsis.com 서울 뉴시스 홍지은 기자 정세균 국무총리는 1일 집중호우로 침수 피해가 발생한 대전 코스모스아파트의 피해 및 복구 상황을 점검했다. 현장 관계자들을 격려하며 재발 방지책 마련을 주문했다. 이 아파트는 256세대 가운데 D동과 E동 28세대가 물에 잠겼고 지상주차장에 주차됐던 차량 100여대 가량도 침수되는 등 피해가 극심한 곳이다. 정 총리는 현장 통합지원본부를 방문해 서철모 대전시 행정부시장으로부터 집중호우 피해 및 복구 현황을 보고받았다.정 총리는 이번 집중호우로 많은 이재민이 발생하고 힘든 시민들이 많이 생긴 것에 있어 진심으로 위로의 말씀을 드린다 고 말했다.이어 어려울 때는 항상 지자체 공직자들의 수고가 많고 또 여러 단체에서 자원봉사를 하기 때문에 고통을 나누어 가질 수 있다는 것이 대한민국의 자부심이 아닐까 생각한다 고 했다. 서울 뉴시스 김명원 기자 정세균 국무총리가 1일 오전 대전 서구 코스모스 아파트를 방문해 자원봉사자들을 격려하고 있다. 사진 총리실 제공 2020.08.01. photo newsis.com또 사고가 나거나 재해가 발생했을 때 그때그때 땜질식으로 처리할 일이 아니고 미리 예방하는 것이 최선 이라고 강조했다.그러면서 예방이 안 돼 재난을 당했다고 하더라도 재난을 당했을 때 임시방편이 아닌 항구적 대책을 세우는 노력이 필요하다 며 앞으로 이런 일이 재발하지 않도록 확실한 대책을 세워달라 고 당부했다.,https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=100&sid2=264&oid=003&aid=0009998896
2020.08.01. 오전 11:33,정치,청와대,감사위원 임명 靑감사원 신경전…월성 원전 갈등 심화,최재형 감사원장 靑 김오수 전 차관 제청 요구 거부대신 제청한 판사 출신 측근 다주택자로 靑 부적합 감사위원회 월성 1호기 조기 폐쇄 타당성 결론 예정文정부 원전 정책 결부돼 靑 감사원장 대립 추이 주목 서울 뉴시스 김진아 기자 최재형 감사원장이 29일 서울 여의도 국회에서 열린 법제사법위원회 전체회의에서 의원의 질의에 답하고 있다. 2020.07.29. bluesoda newsis.com 서울 뉴시스 안채원 홍지은 기자 감사위원 자리를 놓고 최재형 감사원장과 청와대 간 갈등이 수면 위로 드러나고 있다. 9개월째 지연되고 있는 월성 원전 1호기 조기 폐쇄 타당성 검사 발표와 관련해 신경전이 표면화하는 것으로 보인다. 여권 관계자들의 말을 종합해보면 청와대는 이준호 전 감사위원의 퇴임으로 4개월째 공석인 자리에 김오수 전 법무부 차관을 사실상 낙점했던 것으로 전해졌다. 청와대가 최 원장에게 김 전 차관을 감사위원으로 제청할 것을 요구했으나 최 원장은 친여 인사 라는 이유로 여러차례 거부한 것으로 알려졌다.대신 최 원장은 판사 시절 함께 근무한 현직 판사 A씨를 감사위원으로 제청했다. 그러나 청와대는 A씨에 대해 부적합 판정을 내렸다. 한 관계자는 뉴시스와 통화에서 다주택 등의 문제가 있었던 것으로 알고 있다 고 전했다. 실제 공직자 재산신고 내역을 보면 A씨는 서울 서초구를 비롯해 수도권 등에 아파트 5채를 보유하고 있었다.감사위원 임명 논란이 보도되자 지난 29일 청와대 핵심 관계자는 인사와 관련한 사항은 확인해줄 수 없다 면서도 감사위원 임명권은 대통령에게 있다는 점을 분명히 밝힌다 며 우회적으로 불쾌감을 표시했다. 감사원법에 따르면 감사위원은 감사원장이 제청하고 대통령이 임명한다. 최 원장은 같은 날 국회 법제사법위원회 전체회의에서 공석인 감사위원직을 두고 중립적이고 직무상 독립적으로 할 수 있는 분을 제청하기 위해 현재도 노력하고 있다 고 전했다. 서울 뉴시스 장세영 기자 양이원영 더불어민주당 의원이 18일 서울 여의도 국회 소통관에서 월성원전이주대책위 환경운동연합 원자력안전과미래 등의 단체들과 함께 기자회견을 열고 월성 원전 1호기의 폐쇄 과정보다 위법한 수명 연장 과정을 먼저 감사해야 한다고 주장하고 있다. 2020.06.18. photothink newsis.com청와대와 최 원장이 한 석의 감사위원을 두고 팽팽하게 대립하는 배경에는 월성 1호기 조기폐쇄 관련 감사가 있다는 해석이다.감사원의 감사 사항은 감사원 최고의결기구인 감사위원회에서 최종 결정한다. 감사원장을 포함한 총 7명의 위원들로 구성돼있다. 이중 한 석이 현재 공석이다.현재 감사위원회는 2018년 한국수력원자력 한수원 월성 원전 1호기 조기 폐쇄 결정의 타당성을 검토 중이다. 지난 2월이 최종 조사 발표 기한이었지만 9개월째 결론을 내지 못하고 있다. 지난 4월 사흘 연속으로 감사위원회를 열어 월성 원전 감사 보고서 의결을 시도했지만 결국 보류했다. 감사위원들이 월성 1호기 폐쇄에 경제성이 있다는 감사 보고서 의결에 부정적인 입장을 보였는데 최 원장은 의결을 추진하기 위해 감사위원회를 연달아 소집한 것으로 전해지면서 공정성 논란이 제기됐다. 서울 뉴시스 장세영 기자 미래통합당 박형수 김석기 이채익 의원과 이태규 국민의당 의원이 18일 서울 여의도 국회 소통관에서 원자력 정책연대 에너지시민단체 관계자들과 함께 월성1호기 조기폐쇄 반대 기자회견을 하고 있다. 2020.06.18. photothink newsis.com이런 상황에서 나머지 한 명의 감사위원을 탈원전 정책 기조 를 가진 이번 정부에서 추천한 인사로 채워지는 데 최 원장이 부정적 입장을 보였을 것이란 분석이다.한편 최 원장은 지난 29일 오후 국회 법제사법위원회 전체회의에서 대통령의 득표율을 들어서 국정과제의 정당성을 폄훼하려는 의도는 전혀 없었다 고 관련 발언을 해명했다. 여당은 감사원장으로서 부적절한 처신을 했다 며 사퇴까지 요구하고 있다. 청와대는 특별한 언급 없이 직접적 반응은 자제하고 있다.,https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=100&sid2=264&oid=003&aid=0009998776
```

