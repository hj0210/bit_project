# 뉴스 크롤링

## Day2

- csv 파일에 담기 위한 형태는 생성됨
- 여러날짜의 데이터를 수집하기위해 반복문을 사용하여 함수를 생성시켜야 함
- 현재 selenium과 BeautifulSoup을 같이 사용해서 만든 크롤링 데이터가 없음

```python
from selenium import webdriver
from bs4 import BeautifulSoup
import re
import pandas as pd
import csv


class ArticleParser(object):
    special_symbol = re.compile('[\{\}\[\]\/?,;:|\)*~`!^\-_+<>@\#$&▲▶◆◇◀■【】\\\=\(\'\"]')
    content_pattern = re.compile('본문 내용|TV플레이어| 동영상 뉴스|flash 오류를 우회하기 위한 함수 추가function flash removeCallback|tt|앵커 멘트|xa0')

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



URL = 'https://news.naver.com/main/read.nhn?mode=LS2D&mid=shm&sid1=101&sid2=259&oid=421&aid=0004945097'
driver = webdriver.Chrome('D:\\git\\source\\chromedriver\\chromedriver.exe')
driver.implicitly_wait(3)
result = driver.get(URL)
html = driver.page_source
soup = BeautifulSoup(html, 'html.parser')




title_tag = soup.find('h3', {'class': 'tts_head'})
title = ''
title = title + ArticleParser.clear_headline(str(title_tag.text))

write_date_tag = soup.find('span', {'class': 't11'})

main_text_tag = soup.find('div', {'class': '_article_body_contents'})
# 불필요한 태그 제거
for main_text in soup.find_all('script'):
    main_text.extract()
main_text = ''
main_text = main_text + ArticleParser.clear_content(str(main_text_tag.text))


for info in soup:
    temp = []
    write_date = write_date_tag.text
    temp.append(write_date)
    temp.append(title)
    temp.append(main_text)



driver.quit()

print(temp)
```

```shell
['2020.10.22. 오후 11:45', '옵티머스 연루에 KCA 진땀…구글은 대놓고 한국 무시종합2보', '국감초점 서 전 원장 부실투자 송구…권력자들은 알지도 못해 구글 지사장 불출석에 권한 위임도 거절… 인앱결제 막으면 이용자 불리 으름장22일 국회 과학기술정보방송통신위원회 전체회의실에서 열린 종합국정감사에서 이원욱 위원장이 의사봉을 두드리고 있다. 2020.10.22 뉴스1 © News1 성동훈 기자 서울 뉴스1 강은성 기자 손인해 기자 김정현 기자 김승준 기자 펀드환매 사기로 논란이 일고 있는 옵티머스자산운용 의 상품에 투자했던 방송통신전파진흥원 KCA 이 22일 실시된 종합감사에서 야당의 질문공세를 받아내며 진땀을 뺐다. 관심을 모았던 핵심 당사자 최남용 전파진흥원 경인본부장은 이날 국회에 출석하지 않았다. 구글은 4년 연속 국회에 나왔지만 구글코리아 실질 대표도 한국 영업담당 존 리도 아닌 임재현 전무가 출석했다. 구글은 대표 불출석 뿐만 아니라 권한 위임조차 하지 않은채 인앱결제 규제에 대해 이용자와 개발자에게 책임을 지우기 위해 비즈니스 모델 BM 을 바꾸게 될 것 이라며 협박성 발언을 하기도 했다. 부실투자 송구하지만 권력개입 없었다 22일 국회 과학기술정보방송통신위원회의 과학기술정보통신부에 대한 종합감사는 시작부터 옵티머스 논란 으로 야당의 공세가 거셌다. 이날 국감장에는 펀드환매 사기로 논란이 일고 있는 옵티머스자산운용 과 연루설이 제기된 최남용 전파진흥원 경인본부장이 국회 증언대에 설 예정이었다. 최 본부장은 옵티머스 펀드 투자 당시 책임자인 기금운용본부장을 맡고 있었다. 하지만 최 본부장은 감사 하루 전인 지난 21일 불출석 사유서 를 제출하고 이날 모습을 보이지 않았다. 최 본부장은 기금운용본부장으로 재직할 당시 옵티머스 펀드에 투자가 시작되는 시점에서 옵티머스 경영진과 해외여행을 함께 간 사실이 확인되는 등 비위행위 의심도 받고 있는데 불출석사유서를 제출하자 야당 의원들의 질타가 이어졌다. 옵티머스 관련 증인으로 채택된 서석진 전 전파진흥원장은 증언대에 서서 야당의 집중포화를 고스란히 받아냈다. 서석진 전 전파진흥원장이 22일 국회에서 열린 과학기술정보방송통신위원회 국정감사에 증인으로 출석해 사모펀드 옵티머스 투자와 관련한 질의를 받고 답변하고 있다. 2020.10.22 뉴스1 © News1 성동훈 기자서 전 원장은 개별투자에 직접 관여하지 않았더라도 원장으로서 기관장으로서 기관 전체를 대표하는 책임이 있다고 생각하고 송구하게 생각한다 고 사과했다. 서 원장은 옵티머스 투자 결정에 일체 관여하지 않았다는 점을 거듭강조함과 동시에 권력 개입 가능성에도 철저하게 선을 그었다. 그는 금융사기와 관련된 사람들 그리고 임종석 전 청와대 비서실장과 관련된 학교 동문이 보도에 나오기도 했는데 저는 거기에 거론된 모든 분들을 제 인생에서 만난 적도 없고 연락한 적도 없다 고 했다. 지금 협박하냐 …구글 태도에 여야의원들 분노구글은 4년 연속 국감장에 모습을 드러냈다. 그런데 이번에 온 인물은 그간 국회를 찾았던 한국 영업담당 존 리씨가 아니었다. 당초 존 리씨는 구글코리아 대표 자격으로 국회에 온 줄 알았지만 구글코리아의 실제 대표이사는 낸시 메이블 워커라는 미국인이었다.워커 대표는 코로나19 상황에서 이동이 어려워 불출석한다는 사유서는 제출했지만 워커 대표 대신 국감장에 출석하는 임원에게 권한을 위임한다는 위임장은 보내지 않았다. 결국 이날 출석한 임재현 구글코리아 전무는 대부분의 대답을 말씀드릴 권한이 없다 잘 모른다 는 식으로 반복했다. 더구나 구글은 인앱결제와 관련해 국회와 한국 국민을 위협하는 듯한 태도를 취해 의원들의 질타를 듣기도 했다. 임재현 전무는 이날 우리나라 규제당국의 제재와 관련해 이용자들과 개발자들에게 책임을 지키기 위해 비즈니스모델 BM 에 대해 한번 더 생각해야 하지 않을까 하는 우려를 갖고 있다 고 말했다.이에 네이버 부사장 출신인 윤영찬 의원은 개발자와 소비자에게 문제가 발생할 수 있다 BM이 바뀔 수 있다는 협박 아니냐 고 따져 물었다.같은 당 한준호 의원도 BM을 바꾸고 책임이 국민에게 전가될 거라는 건 겁박도 아니고 무슨 내용이냐 며 인앱결제 방지법이 통과되면 결국 이용자나 개발사에 전이시키겠다는 거냐 고 했다.이에 임 전무는 그건 아니 라고 몸을 낮추면서도 구글플레이는 방대한 플랫폼이고 유지하기 위한 개발비용이 필요하다 고 했다.임재현 구글코리아 전무가 22일 오후 서울 여의도 국회에서 열린 국회 정무위원회의 국무조정실 등에 대한 2020년도 국정감사에서 증인으로 출석해 질의에 답변하고 있다. 2020.10.22 뉴스1 © News1 성동훈 기자김상희 민주당 의원은 미국 법무부가 구글을 반독점법 위반 혐의로 제소한 사실을 거론하며 과기정통부가 구글의 자사앱 선탑재에 대해 면밀히 조사하고 조치를 취해야 한다고 생각한다 며 시장 지배력을 과시해서 국내 시장과 소비자를 기만해선 안 된다 고 지적했다.최기영 과학기술정보통신부 장관은 이에 대해 우월적 지위 남용이 될 수 있다는 생각이 든다 며 또 혹시나 다른 앱스토어에 등록을 못하게 한다든가 이런 불공정 거래가 없길 바란다. 공정위·방통위와 협력하고 있다 고 했다.']
```

