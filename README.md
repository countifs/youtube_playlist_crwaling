# youtube_playlist_crwaling
유튜브 재생목록을 노션 또는 티스토리(블로그)에 복사-붙여넣기 위한 목적으로 `BeautifulSoup`을 이용하여 Youtube playlist를 크롤링하였고, `openpyxl`을 이용하여 제목에 url링크를 하이퍼링크로 연동한 결과를 엑셀 파일로 저장하는 코드입니다.

## 1. 유튜브 재생목록 크롤링

```python
# url 주소 입력
url = input('유튜브 재생목록 주소를 입력하세요: ')

# 재생목록 제목에서 제거할 텍스트(문자)를 입력
rep_text  = ''

# 웹 브라우저 작동을 위한 라이브러리
from selenium import webdriver
from urllib.request import urlopen
from bs4 import BeautifulSoup
from urllib.parse import quote_plus
from selenium.webdriver.common.keys import Keys

# 웹브라우저 작동을 기다리기 위한 라이브러리
import time
import random

# 시간 관련 라이브러리
from datetime import datetime, timedelta
from pytz import timezone

# IPython
from IPython.display import display

# 경고 무시
import warnings
warnings.filterwarnings(action='ignore')

# 데이터프레임 및 CSV 파일 저장을 위한 라이브러리
import pandas as pd

# 데이터프레임 출력
from tabulate import tabulate

# 크롬드라이버 option설정
options = webdriver.ChromeOptions()
options.add_argument('--headless')        # Head-less 설정
options.add_argument('--no-sandbox')
options.add_argument('--disable-dev-shm-usage')

def get_playlist(url, rep_text):

    # 브라우저 생성
    driver = webdriver.Chrome('chromedriver', options=options)

    # 웹사이트 열기
    driver.get(url)

    # 로딩이 끝날 때까지 2초 정도 기다림
    driver.implicitly_wait(2)

   # 안정적인 페이지 소스 추출을 위해 3초 정도 기다림
    time.sleep(3)

    # 페이지 소스 추출
    global html_source, soup_source
    html_source = driver.page_source
    soup_source = BeautifulSoup(html_source, 'lxml')

    # 파싱정보 가져오기
        
    parsing = soup_source.find_all('a', class_ = 'yt-simple-endpoint style-scope ytd-playlist-video-renderer')
    video_time = soup_source.find_all('span', class_ ='style-scope ytd-thumbnail-overlay-time-status-renderer') #검색했을 때 검색숫자가 안맞아서 확인이 필요함
    
    global name_list, url_list, time_list
        
    # 파싱정보 정리하기
    name_list = []
    url_list = []
    time_list = []

    for i in range(len(parsing)):
        name_list.append(parsing[i].text.strip())
        main = 'https://www.youtube.com'
        sub = parsing[i].get('href')  
        url_list.append(f'{main}{sub}')
        time_list.append(video_time[i].text.strip())

   # 출력용 데이터 프레임 구성하기     
    playlist = {
        '제목' : name_list,
        '시간' : time_list,
        'URL' : url_list, 
    }
   # 제목에서 제거할 문자 변환하기
    playlist = pd.DataFrame(playlist)
    playlist['제목'] = playlist['제목'].apply(lambda x: x.replace(rep_text,'').strip())
    
    return playlist

final_result = get_playlist(url, rep_text)
final_result.index = final_result.index + 1
final_result
```

<br>


## 2. openpyxl 하이퍼링크 기능 및 styles를 사용하여 엑셀 파일로 저장

```python
file_name = input("저장할 파일 이름을 입력하세요: ")

from openpyxl import Workbook
from openpyxl.styles import Border, Side

# URL and corresponding name lists

# Create a new workbook and worksheet
wb = Workbook()
ws = wb.active

# 테두리 스타일 지정
thin_border = Border(left=Side(style='thin'), 
                     right=Side(style='thin'), 
                     top=Side(style='thin'), 
                     bottom=Side(style='thin'))

list_title = soup_source.find(class_ = 'style-scope yt-dynamic-sizing-formatted-string yt-sans-28').text
channel_name = soup_source.find('a', class_ = 'yt-simple-endpoint style-scope yt-formatted-string').text
title = channel_name + "-" + list_title

name_header_cell = ws.cell(row=1, column=1, value=title)  # Insert name header into cell
name_header_cell.border = thin_border
name_header_cell.font = Font(bold=True)

time_header_cell = ws.cell(row=1, column=2, value="시간")  # Insert URL header into cell
time_header_cell.border = thin_border
time_header_cell.font = Font(bold=True)

# Iterate through the URL and name lists, adding each URL to a new row
for url, name, time in zip(url_list, name_list, time_list):
    row = ws.max_row + 1  # Next row number
    name_cell = ws.cell(row=row, column=1, value=name)  # Insert name into cell
    name_cell.hyperlink = url  # Add hyperlink to cell
    name_cell.style = "Hyperlink"  # Set cell style to "Hyperlink"
    name_cell.border = thin_border  # Add border to cell
    
    time_cell = ws.cell(row=row, column=2, value=time)  # Insert URL into cell
    time_cell.border = thin_border  # Add border to cell

# 셀의 너비를 셀의 내용에 맞게 자동 조정
for column in ws.columns:
    max_length = 0
    column = get_column_letter(column[0].column)  # Get the column name
    for cell in ws[column]:  # Iterate through all cells in the column
        try:  # Check if the cell contains text
            if len(str(cell.value)) > max_length:
                max_length = len(cell.value)
        except:
            pass
    adjusted_width = (max_length + 10)
    ws.column_dimensions[column].width = adjusted_width


# Save changes and create Excel file
wb.save(f'{file_name}.xlsx')
```
