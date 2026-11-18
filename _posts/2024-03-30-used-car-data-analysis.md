---
title: "Used Car Data Analysis and Visualization"
date: 2024-03-30
permalink: /blog/used-car-data-analysis/
excerpt: "A hands-on used-car analysis project using Python web scraping, big-data tools, SQL-based aggregation, and HTML/ECharts visualization."
categories:
  - Data Analysis
tags:
  - Python
  - Hadoop
  - Spark
  - ECharts
---

> An English edition of my [original used-car data analysis project on CSDN](https://blog.csdn.net/weixin_75027717/article/details/137185817), archived under **March 2024**. This article records a historical learning project. The source code retains its original placeholders and the charts describe the collected sample, not current market-wide statistics.

# Introduction

China's used-car market has grown as car ownership and vehicle turnover have increased. As vehicles age, maintenance needs and operating costs can motivate owners to replace them. This project investigates used-car listings by collecting and analyzing data, then presenting the results visually.

# 1. Development environment

## 1.1 JDK

Copy the Java directory to `C:\ProgramFiles`. Configure the system environment:

```text
CLASSPATH=.;C:\ProgramFiles\Java\jdk1.8.0_131\lib
Path: C:\ProgramFiles\Java\jdk1.8.0_131\bin
```

These are the original Windows paths; use the actual installation paths on your machine.

## 1.2 Maven

Place Maven under `C:\ProgramFiles`, then add the following directory to `Path`:

```text
C:\ProgramFiles\apache-maven-3.6.3\bin
```

Check the installation with `mvn -v`.

## 1.3 IntelliJ IDEA

Install IntelliJ IDEA, choosing a location without spaces or Chinese characters as recommended in the original notes. The source also mentions “crack file configuration”; that portion is not reproduced as an installation step. Use an appropriately licensed or free/community edition instead.

## 1.4 Scala

Within IntelliJ IDEA, navigate to **File → Settings → Plugins**, search for **Scala**, and install the plugin.

![Installing the Scala plugin](/images/blog/used-car/aac4f40470bb13fa956988d624e7d3eb.png)

## 1.5 VMware, Ubuntu and Hadoop

This project also documents a virtualized Hadoop 2.10.2 environment. First install **VMware Workstation 17** and **Ubuntu 16**.

![VMware and Ubuntu setup 1](/images/blog/used-car/0761cad12be2fca6ad5e8118ae9a29f2.png)

![VMware and Ubuntu setup 2](/images/blog/used-car/57e2d270afb092f6c8da01105e0970ed.png)

Log in to Ubuntu and create the `hadoop` user. The source used the following steps:

```bash
sudo passwd root
sudo useradd -m hadoop -s /bin/bash
sudo passwd hadoop
sudo adduser hadoop sudo
su hadoop
```

The original notes show root and user prompts; the commands above are formatted consistently for readability.

Install/configure the JDK and verify it:

```bash
java -version
```

![JDK setup and version test 1](/images/blog/used-car/612afa9e2e684d21f104e058984413b7.png)

![JDK setup and version test 2](/images/blog/used-car/a5296146f24ff42d85ef694356e429a5.png)

Download Hadoop from the [Apache Hadoop website](https://hadoop.apache.org/) using Ubuntu's Firefox browser. The source uses **Hadoop 2.10.2**:

```bash
mv hadoop-2.10.2.tar.gz /home/hadoop
tar -xvf hadoop-2.10.2.tar.gz
```

The extracted directory is `/home/hadoop/hadoop-2.10.2/`.

![Downloading and extracting Hadoop 1](/images/blog/used-car/53b32010baab07618006373fa7a295b3.png)

![Downloading and extracting Hadoop 2](/images/blog/used-car/155356ce326999892c021b41ad5f1588.png)

Configure the environment using `/etc/profile` and reload it:

```bash
sudo vim /etc/profile
source /etc/profile
hadoop version
```

The source supplies the detailed profile values in the screenshot rather than as copyable text.

![Environment profile and Hadoop version output 1](/images/blog/used-car/405c995b71be9122b1159773ca1fc643.png)

![Environment profile and Hadoop version output 2](/images/blog/used-car/68903b4bf20ed656902dbbb5702d328b.png)

![Environment profile and Hadoop version output 3](/images/blog/used-car/a576ad4d5f63ccdfd72767f2643acc07.png)

# 2. Project design

The original project covers four connected activities:

1. **Data collection:** scrape structured used-car listing information with Python.
2. **Data processing:** organize the collected data into usable tables or CSV/JSON formats.
3. **Data analysis:** calculate listing counts and relationships between vehicle age, mileage, price and city.
4. **Visualization:** display analytical results through HTML and chart components such as ECharts.

# 3. Implementation

## 3.1 Collecting vehicle listings with Python

The following is the **original illustrative script**, reformatted without replacing its placeholder domain, Cookie or User-Agent. Those placeholders mean it is **not immediately runnable**. For real collection, comply with website rules, robots policies and applicable privacy requirements.

```python
import csv

import requests
import parsel

f = open('datacar.csv', mode='a', encoding='utf-8', newline='')
csv_writer = csv.DictWriter(f, fieldnames=[
    '标题',
    '年份',
    '里程',
    '城市',
    '价格',
    '标签儿',
    '保修又没',
    '详情页',
])
csv_writer.writeheader()

for page in range(1, 51):
    print(f'=======ciajidi{page}yedeshujuneiron======')
    headers = {
        'Cookie': '......',
        'User-Agent': '......'
    }
    url = f'https://..........com/all/?page={page}&'
    response = requests.get(url=url, headers=headers)
    html_data = response.text
    print(response)
    selector = parsel.Selector(html_data)
    lis = selector.css('.Content_left .gongge_ul li')

    for li in lis:
        title = li.css('.title::attr(title)').get()
        car_info = li.css('.gongge_main p i::text').getall()
        year = car_info[0]
        mileage = car_info[1]
        city = car_info[2].strip()
        label = li.css('.car_tag em::text').get().strip()
        tag = li.css('.tc_label::text').get()
        price = li.css('.Total::text').get()
        href = li.css('.title::attr(href)').get()

        dit = {
            '标题': title,
            '年份': year,
            '里程': mileage,
            '城市': city,
            '价格': price,
            '标签儿': label,
            '保修又没': tag,
            '详情页': href,
        }
        csv_writer.writerow(dit)
        print(title, year, mileage, city, price, label, tag, href)
```

The collected fields are the vehicle title, year, mileage, city, price, listing tags, warranty label and detail-page URL. The original script writes these records to `datacar.csv`. It assumes a particular page structure and nonempty selector results, so an actual implementation would require appropriate error handling, selectors and credentials.

## 3.2 Functional design and visualization

### Analysis 1: Used-car listing trends by year

**Input:** `year`  
**Output:** `year`, `count`  
**Visualization:** bar chart

Count vehicles within the selected year range. The original implementation describes using SQL to aggregate data and converting the results to JSON for HTML/chart rendering.

**Data processing and analysis code screenshots:**

![Yearly listing aggregation 1](/images/blog/used-car/f53730f75a902fb331288b78c135b8c0.png)

![Yearly listing aggregation 2](/images/blog/used-car/b4cda0b8ff3faa06f14f91efc1d0e734.png)

**HTML visualization code:**

![Yearly trend visualization code](/images/blog/used-car/736989b5ce2ca22c10d64c86d09b07b7.png)

**Visualization result:**

![Used-car trend bar chart](/images/blog/used-car/042021f3e170e55b5eb0f65d98f49037.png)

**Original interpretation:** The observed sample peaks around **2021**, followed by a decline. The original author speculated that pandemic-related circumstances could have affected this trend. This is a sample-level observation, not proof of a market-wide causal relationship; listing counts are also not necessarily completed vehicle sales.

### Analysis 2: Mileage and vehicle price

**Input:** `mileage`, `price`  
**Output:** `mileage`, `price`  
**Visualization:** line chart

Read the mileage and price fields, order or group records by mileage, and prepare the results using SQL/JSON for visualization.

**Data-processing code:**

![Mileage and price analysis code 1](/images/blog/used-car/5e206aa8fa5c000371f319b50bac9111.png)

![Mileage and price analysis code 2](/images/blog/used-car/8a3c179b4c72757cdfead7fe053318b4.png)

**HTML chart code:**

![Mileage-price visualization implementation](/images/blog/used-car/a22abd2ce67d8ebdd62f4eca1434841d.png)

**Visualization result:**

![Mileage and vehicle price chart](/images/blog/used-car/b21d6b428864090727bd92115eb0c6dd.png)

**Original interpretation:** The sample's lowest average listed price appeared in the **80,000–100,000 km** interval, while the highest appeared in **0–20,000 km**. Vehicle model, model year and brand may also influence prices, so this should not be interpreted as a controlled estimate of mileage's causal effect.

### Analysis 3: Preferred vehicle models by city

**Input:** `city`, `title`, `count`  
**Output:** `city`, `title`, `count`  
**Visualization:** 3D bar chart

Aggregate the number of vehicles for each city/model combination and transform SQL results into JSON for the chart.

**Data-processing code:**

![City and vehicle model aggregation 1](/images/blog/used-car/7cf732455427a7d7f67a07b2c80e0dde.png)

![City and vehicle model aggregation 2](/images/blog/used-car/b300f08068a56a617477b0461d94e547.png)

**HTML visualization code:**

![3D chart implementation 1](/images/blog/used-car/9a8b7982c9c2e25a5456adab502fe171.png)

![3D chart implementation 2](/images/blog/used-car/8566528a20c5d53ecff2957b6be37432.png)

![3D chart implementation 3](/images/blog/used-car/fed62eea8a7da18d887d8e2c633a7c0c.png)

**Visualization result:**

![City and brand distribution 3D chart](/images/blog/used-car/352dfe7cc4c9e2d8d9b55ab17313086e.png)

**Original interpretation:** Volkswagen appeared most frequently across the cities in the analyzed listings. Listing frequency does not establish completed sales volume or consumer preferences for the wider population.

### Analysis 4: Additional web data collection

The source also documents collecting listings from [Taoche's Zhengzhou site](https://zhengzhou.taoche.com/) using Python in PyCharm. The code and resulting console output are preserved in the screenshots.

![Additional Python collection code and output 1](/images/blog/used-car/5d8b4ec0190b61f4b820650d2f5d975d.png)

![Additional Python collection code and output 2](/images/blog/used-car/8c6e2c52f634d2b50c19dde668d7f22e.png)

### Analysis 5: Local keyword search

Convert collected records to **JSON**, load them into an HTML page and search by keyword to retrieve matching car information.

**Implementation code:**

![JSON search and HTML implementation 1](/images/blog/used-car/fb7e5f61fa7483ff2c4a31e14cb276f1.png)

![JSON search and HTML implementation 2](/images/blog/used-car/96ec6a3a6a139d52e26c26a9ca564bd6.png)

![JSON search and HTML implementation 3](/images/blog/used-car/066cb652e37ed64d02bf4001c22f7454.png)

**HTML interface:**

![Local used-car search interface](/images/blog/used-car/5f62cf94baee16309642e2d0b505e7fb.png)

# 4. Conclusion

This project brought together **IntelliJ IDEA, HTML, Spark and ECharts** with Python data collection and analysis. It provided hands-on experience with environment configuration, web scraping, data processing, SQL-based aggregation, interactive charts and searching local JSON data. The interpretations above describe the project's historical listing sample, rather than the current used-car market.

---

*Original article: [Used-car data analysis on my CSDN blog](https://blog.csdn.net/weixin_75027717/article/details/137185817). The source's code, screenshots and historical software versions are retained where available; prose and commentary have been translated into English.*