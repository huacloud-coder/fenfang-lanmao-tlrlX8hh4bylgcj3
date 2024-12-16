
## 1\. **背景**


这段时间项目比较忙，所以本qiang\~有些耽误了学习，不过也算是百忙之中，抽取时间来支撑一个读者的需求，即爬取一些财经网站的新闻并自动聚合。


该读者看了之前的《[AI资讯的自动聚合及报告生成](https://github.com)》文章后，想要将这一套流程嵌套在财经领域，因此满打满算耗费了2\-3天时间，来完成了该需求。


注意：爬虫不是本人的强项，只是一丢丢兴趣而已; 其次，本篇文章主要是用于个人学习，客官们请勿直接商业使用。


## 2\. **面临的难点**


1\. 爬虫框架选取: 采用之前现学现用的crawl4ai作为基础框架，使用其高阶技能来逼近模拟人访问浏览器，因为网站都存在反爬机制，如鉴权、cookie等；


2\. 外网新闻: 需要kexue上网；


3\. 新闻内容解析: 此处耗费的工作量最多，并不是html的页面解析有多难，主要是动态页面加载如何集成crawl4ai来实现，且每个新闻网站五花八门。


## 3\. **数据源**




| 数据源 | url | 备注 |
| --- | --- | --- |
| 财lian社 | [https://www.cls.cn/depth?id\=1000](https://github.com) [https://www.cls.cn/depth?id\=1003](https://github.com) [https://www.cls.cn/depth?id\=1007](https://github.com) | 1000: 头条, 1003: A股, 1007: 环球 |
| 凤huang网 | [https://finance.ifeng.com/shanklist/1\-64\-/](https://github.com) |  |
| 新lang | [https://finance.sina.com.cn/roll/\#pageid\=384\&lid\=2519\&k\=\&num\=50\&page\=1](https://github.com) [https://finance.sina.com.cn/roll/\#pageid\=384\&lid\=2672\&k\=\&num\=50\&page\=1](https://github.com):[veee加速器](https://blog.liuyunzhuge.com) | 2519: 财经 2672: 美股 |
| 环qiu时报 | https://finance.huanqiu.com |  |
| zaobao | [https://www.zaobao.com/finance/china](https://github.com) [https://www.zaobao.com/finance/world](https://github.com) | 国内及世界 |
| fox | [https://www.foxnews.com/category/us/economy](https://github.com) [https://www.foxnews.com//world/global\-economy](https://github.com) | 美国及世界 |
| cnn | https://edition.cnn.com/business https://edition.cnn.com/business/china | 国内及世界 |
| reuters | https://www.reuters.com/business |  |


## 4\. **部分源码**


为了减少风险，本qiang\~只列出财lian社网页的解析代码，读者如想进一步交流沟通，可私信联系。


代码片段解析:


1\. schema是以json格式叠加css样式的策略，crawl4ai基于schema可以实现特定元素的结构化解析


2\. js\_commands是js代码，主要用于模拟浏览新闻时的下翻页 




```
import asyncio
from crawl4ai import AsyncWebCrawler
from crawl4ai.extraction_strategy import JsonCssExtractionStrategy
import json
from typing import Dict, Any, Union, List
import os
import datetime
import re
import hashlib


def md5(text):
    m = hashlib.md5()
    m.update(text.encode('utf-8'))
    return m.hexdigest()


def get_datas(file_path, json_flag=True, all_flag=False, mode='r'):
    """读取文本文件"""
    results = []
    
    with open(file_path, mode, encoding='utf-8') as f:
        for line in f.readlines():
            if json_flag:
                results.append(json.loads(line))
            else:
                results.append(line.strip())
        if all_flag:
            if json_flag:
                return json.loads(''.join(results))
            else:
                return '\n'.join(results)
        return results
    

def save_datas(file_path, datas, json_flag=True, all_flag=False, with_indent=False, mode='w'):
    """保存文本文件"""
    with open(file_path, mode, encoding='utf-8') as f:
        if all_flag:
            if json_flag:
                f.write(json.dumps(datas, ensure_ascii=False, indent= 4 if with_indent else None))
            else:
                f.write(''.join(datas))
        else:
            for data in datas:
                if json_flag:
                    f.write(json.dumps(data, ensure_ascii=False) + '\n') 
                else:
                    f.write(data + '\n')


class AbstractAICrawler():
    
    def __init__(self) -> None:
        pass
    def crawl():
        raise NotImplementedError()


class AINewsCrawler(AbstractAICrawler):
    def __init__(self, domain) -> None:
        super().__init__()
        self.domain = domain
        self.file_path = f'data/{self.domain}.json'
        self.history = self.init()
    
    def init(self):
        if not os.path.exists(self.file_path):
            return {}
        return {ele['id']: ele for ele in get_datas(self.file_path)}
    
    def save(self, datas: Union[List, Dict]):
        if isinstance(datas, dict):
            datas = [datas]
        self.history.update({ele['id']: ele for ele in datas})
        save_datas(self.file_path, datas=list(self.history.values()))
    
    async def crawl(self, url:str, 
                    schema: Dict[str, Any]=None, 
                    always_by_pass_cache=True, 
                    bypass_cache=True,
                    headless=True,
                    verbose=False,
                    magic=True,
                    page_timeout=15000,
                    delay_before_return_html=2.0,
                    wait_for='',
                    js_code=None,
                    js_only=False,
                    screenshot=False,
                    headers={}):
        
        extraction_strategy = JsonCssExtractionStrategy(schema, verbose=verbose) if schema else None
        
        async with AsyncWebCrawler(verbose=verbose, 
                                   headless=headless, 
                                   always_by_pass_cache=always_by_pass_cache, headers=headers) as crawler:
            result = await crawler.arun(
                url=url,
                extraction_strategy=extraction_strategy,
                bypass_cache=bypass_cache,
                page_timeout=page_timeout,
                delay_before_return_html=delay_before_return_html,
                wait_for=wait_for,
                js_code=js_code,
                magic=magic,
                remove_overlay_elements=True,
                process_iframes=True,
                exclude_external_links=True,
                js_only=js_only,
                screenshot=screenshot
            )

            assert result.success, "Failed to crawl the page"
            if schema:
                res = json.loads(result.extracted_content)
                if screenshot:
                    return res, result.screenshot
                return res
            return result.html


class FinanceNewsCrawler(AINewsCrawler):
    
    def __init__(self, domain='') -> None:
        super().__init__(domain)
    
    def save(self, datas: Union[List, Dict]):
        if isinstance(datas, dict):
            datas = [datas]
        self.history.update({ele['id']: ele for ele in datas})
        save_datas(self.file_path, datas=datas, mode='a')
    
    async def get_last_day_data(self):
        last_day = (datetime.date.today() - datetime.timedelta(days=1)).strftime('%Y-%m-%d')
        datas = self.init()
        return [v for v in datas.values() if last_day in v['date']]
    

class CLSCrawler(FinanceNewsCrawler):
    """
        财某社新闻抓取
    """
    def __init__(self) -> None:
        self.domain = 'cls'
        super().__init__(self.domain)
        self.url = 'https://www.cls.cn'
        
    async def crawl_url_list(self, url='https://www.cls.cn/depth?id=1000'):
        schema = {
            'name': 'caijingwang toutiao page crawler',
            'baseSelector': 'div.f-l.content-left',
            'fields': [
                {
                    'name': 'top_titles',
                    'selector': 'div.depth-top-article-list',
                    'type': 'nested_list',
                    'fields': [
                        {'name': 'href', 'type': 'attribute', 'attribute':'href', 'selector': 'a[href]'}
                    ]
                },
                {
                    'name': 'sec_titles',
                    'selector': 'div.depth-top-article-list  li.f-l',
                    'type': 'nested_list',
                    'fields': [
                        {'name': 'href', 'type': 'attribute', 'attribute':'href', 'selector': 'a[href]'}
                    ]
                },
                {
                    'name': 'bottom_titles',
                    'selector': 'div.b-t-1 div.clearfix',
                    'type': 'nested_list',
                    'fields': [
                        {'name': 'href', 'type': 'attribute', 'attribute':'href', 'selector': 'a[href]'}
                    ]
                }
            ]
        }
        
        js_commands = [
            """
            (async () => {{
                
                await new Promise(resolve => setTimeout(resolve, 500));
                
                const targetItemCount = 100;
                
                let currentItemCount = document.querySelectorAll('div.b-t-1 div.clearfix a.f-w-b').length;
                let loadMoreButton = document.querySelector('.list-more-button.more-button');
                
                while (currentItemCount < targetItemCount) {{
                    window.scrollTo(0, document.body.scrollHeight);
                    
                    await new Promise(resolve => setTimeout(resolve, 1000));
                    
                    if (loadMoreButton) {
                        loadMoreButton.click();
                    } else {
                        console.log('没有找到加载更多按钮');
                        break;
                    }
                    
                    await new Promise(resolve => setTimeout(resolve, 1000));
                    
                    currentItemCount = document.querySelectorAll('div.b-t-1 div.clearfix a.f-w-b').length;
                    
                    loadMoreButton = document.querySelector('.list-more-button.more-button');
                }}
                console.log(`已加载 ${currentItemCount} 个item`);
                return currentItemCount;
            }})();
            """
        ]
        wait_for = ''
        
        results = {}
        
        menu_dict = {
            '1000': '头条',
            '1003': 'A股',
            '1007': '环球'
        }
        for k, v in menu_dict.items():
            url = f'https://www.cls.cn/depth?id={k}'
            try:
                links = await super().crawl(url, schema, always_by_pass_cache=True, bypass_cache=True, js_code=js_commands, wait_for=wait_for, js_only=False)
            except Exception as e:
                print(f'error {url}')
                links = []
            if links:
                links = [ele['href'] for eles in links[0].values() for ele in eles if 'href' in ele]
            links = sorted(list(set(links)), key=lambda x: x)
            results.update({f'{self.url}{ele}': v for ele in links})
        return results
    
    async def crawl_newsletter(self, url, category):
        schema = {
            'name': '财联社新闻详情页',
            'baseSelector': 'div.f-l.content-left',
            'fields': [
                {
                    'name': 'title',
                    'selector': 'span.detail-title-content',
                    'type': 'text'
                },
                {
                    'name': 'time',
                    'selector': 'div.m-r-10',
                    'type': 'text'
                },
                {
                    'name': 'abstract',
                    'selector': 'pre.detail-brief',
                    'type': 'text',
                    'fields': [
                        {'name': 'href', 'type': 'attribute', 'attribute':'href', 'selector': 'a[href]'}
                    ]
                },
                {
                    'name': 'contents',
                    'selector': 'div.detail-content p',
                    'type': 'list',
                    'fields': [
                        {'name': 'content', 'type': 'text'}
                    ]
                },
                {
                    'name': 'read_number',
                    'selector': 'div.detail-option-readnumber',
                    'type': 'text'
                }
            ]
        }
        
        wait_for = 'div.detail-content'
        try:
            results = await super().crawl(url, schema, always_by_pass_cache=True, bypass_cache=True, wait_for=wait_for)
            result = results[0]
        except Exception as e:
            print(f'crawler error: {url}')
            return {}
        
        return {
            'title': result['title'],
            'abstract': result['abstract'],
            'date': result['time'],
            'link': url,
            'content': '\n'.join([ele['content'] for ele in result['contents'] if 'content' in ele and ele['content']]),
            'id': md5(url),
            'type': category,
            'read_number': await self.get_first_float_number(result['read_number'], r'[-+]?\d*\.\d+|\d+'),
            'time': datetime.datetime.now().strftime('%Y-%m-%d')
        }
    
    async def get_first_float_number(self, text, pattern):
        match = re.search(pattern, text)
        if match:
            return round(float(match.group()), 4)
        return 0
    
    async def crawl(self):
        link_2_category = await self.crawl_url_list()
        for link, category in link_2_category.items():
            _id = md5(link)
            if _id in self.history:
                continue
            news = await self.crawl_newsletter(link, category)
            if news:
                self.save(news)
        return await self.get_last_day_data()
    
if __name__ == '__main__':
    asyncio.run(CLSCrawler().crawl())
```


5\. **总结**


一句话足矣\~


开发了一款新闻资讯的自动聚合的工具，基于crawl4ai框架实现。


有问题可以私信或留言沟通！


## 6\. **参考**


(1\) Crawl4ai: [https://github.com/unclecode/crawl4ai](https://github.com)


 ![](https://img2024.cnblogs.com/blog/602535/202412/602535-20241216133332440-608882403.png)


 


