---
layout: post
title: Get movie ranking using python
categories: 
 - python
permalink: /posts/6
nocomments: true  # Disable the comment box. This is EasyBook feature
---

Today, I'm going to record a list about the movie ranking from "https://movie.douban.com/top250". This list contains 6 tags as follows: Name, Director, MainActor, ReleaseDate, Classification and scores.

First, I need to get the HTML text using requests library:

```python
def getHTML(url):
    try:
        headers = {
            'user-agent': 'Mozilla/5.0',
            'Host': 'movie.douban.com'
        }
        r = requests.get(url, headers=headers, timeout=10)
        r.raise_for_status()
        r.encoding = 'utf-8'
        return r.text
    except:
        return 'Spider failed！'
```



Next, Save the list into csvFile by using the method save_columns_to_csv:

```python
def save_columns_to_csv(columns, encoding='utf-8'):

    with open('movie.csv', 'a', encoding=encoding) as csvfile:
        csv_write = csv.writer(csvfile)
        csv_write.writerow(columns)
```



Then, My basic thoughts about this reptile are as follow:

1.Find each movie's url in the front page

2.Use these urls to each movie's detail page to obtain full information

3.Save informations to a csvFile

In these steps, I need to use the re library to find the useful information

```python
for i in range(0, 2):
            url = 'https://movie.douban.com/top250?start=' + str(i * 25)
            data = getHTML(url)
            if(len(data)==0):
                print('Page ' + str(i) + ': failed!')
            else:
                print('Page ' + str(i) + ': success!')
                soup = BeautifulSoup(data, 'lxml')
                List = soup.find_all('div', attrs={'class': 'hd'})
                for each in List:
                    detailUrl = each.a.attrs.get('href')
                    detailMovieList.append(detailUrl)
                for j in detailMovieList:
                    newList = []
                    detaildata = getHTML(j)
                    patName = '<span property="v:itemreviewed">(.*?)</span>'
                    Name = re.compile(patName).findall(detaildata)[0].split(' ')[0]
                    newList.append(Name)

                    patDirector = 'rel="v:directedBy">(.*?)</a>'
                    Director = re.compile(patDirector).findall(detaildata)[0]
                    newList.append(Director)

                    patMainActor = 'rel="v:starring">(.*?)</a>'
                    MainActor = re.compile(patMainActor).findall(detaildata)[0]
                    newList.append(MainActor)

                    patReleaseData = '<span property="v:initialReleaseDate" content="(.*?)">'
                    Releasedata = re.compile(patReleaseData).findall(detaildata)[0].split('-')[0]
                    newList.append(Releasedata)

                    patClassification = '<span property="v:genre">(.*?)</span>'
                    Classification = re.compile(patClassification).findall(detaildata)
                    newList.append(Classification)

                    patScore = '<strong class="ll rating_num" property="v:average">(.*?)</strong>'
                    Score = re.compile(patScore).findall(detaildata)[0]
                    newList.append(Score)

                    save_columns_to_csv(newList)
                    print('Page' + str(i+1) + ': ' + Name + ' data success!')
```

Finally, the csvFile looks like this:

![avatar](https://s6.jpg.cm/2021/11/27/LLIari.png)

Appendix：

```python
import requests
from bs4 import BeautifulSoup
import re
import csv
import os
def getHTML(url):
    try:
        headers = {
            'user-agent': 'Mozilla/5.0',
            'Host': 'movie.douban.com'
        }
        r = requests.get(url, headers=headers, timeout=10)
        r.raise_for_status()
        r.encoding = 'utf-8'
        return r.text
    except:
        return 'Spider failed！'


def save_columns_to_csv(columns, encoding='utf-8'):

    with open('movie.csv', 'a', encoding=encoding) as csvfile:
        csv_write = csv.writer(csvfile)
        csv_write.writerow(columns)


if __name__ == '__main__':
    TitleList = ['Name', 'Director', 'MainActors', 'ReleaseDate', 'classification', 'Score']
    if os.path.exists('movie.csv'):
        os.remove('movie.csv')
    save_columns_to_csv(TitleList)
    detailMovieList = []
    try:
        for i in range(0, 2):
            url = 'https://movie.douban.com/top250?start=' + str(i * 25)
            data = getHTML(url)
            if(len(data)==0):
                print('Page ' + str(i) + ': failed!')
            else:
                print('Page ' + str(i) + ': success!')
                soup = BeautifulSoup(data, 'lxml')
                List = soup.find_all('div', attrs={'class': 'hd'})
                for each in List:
                    detailUrl = each.a.attrs.get('href')
                    detailMovieList.append(detailUrl)
                for j in detailMovieList:
                    newList = []
                    detaildata = getHTML(j)
                    patName = '<span property="v:itemreviewed">(.*?)</span>'
                    Name = re.compile(patName).findall(detaildata)[0].split(' ')[0]
                    newList.append(Name)

                    patDirector = 'rel="v:directedBy">(.*?)</a>'
                    Director = re.compile(patDirector).findall(detaildata)[0]
                    newList.append(Director)

                    patMainActor = 'rel="v:starring">(.*?)</a>'
                    MainActor = re.compile(patMainActor).findall(detaildata)[0]
                    newList.append(MainActor)

                    patReleaseData = '<span property="v:initialReleaseDate" content="(.*?)">'
                    Releasedata = re.compile(patReleaseData).findall(detaildata)[0].split('-')[0]
                    newList.append(Releasedata)

                    patClassification = '<span property="v:genre">(.*?)</span>'
                    Classification = re.compile(patClassification).findall(detaildata)
                    newList.append(Classification)

                    patScore = '<strong class="ll rating_num" property="v:average">(.*?)</strong>'
                    Score = re.compile(patScore).findall(detaildata)[0]
                    newList.append(Score)

                    save_columns_to_csv(newList)
                    print('Page' + str(i+1) + ': ' + Name + ' data success!')
    except Exception as err:
        print(err)
```

