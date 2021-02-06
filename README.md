
> 更多详情见我的博客[Py爬虫+基于echart可视化csv数数据](https://shimmerjordan.site/2020/06/19/pyecharts_tour_statistic/)

>	[前面一篇文章](https://shimmerjordan.site/2020/06/17/py_csv_primary/)谈及了Python基于`matplotlib`模块的csv数据可视化，这篇文章从更广度的数据角度，结合Python爬虫并利用`pyecharts`这个更优雅的数据可视化模块，对从网络爬取的数据集进行可视化处理。由于Echart的可操作性，最终生成的可视化图像以html的形式展现以供检阅，文中也会贴出静态预览图。

<!--more-->

# 1 爬取目标分析

## 1.1 分析目标url

​		这里选用了[去哪儿旅行]([http://piao.qunar.com)作为爬虫的目标，因为观察到点击”门票“后搜索相应关键词，例如热门景点，跳转URL为`http://piao.qunar.com/ticket/list.htm?keyword=辽宁&region=&from=mpl_search_suggest&page={}`， 进一步分析可知，该url的结构为`http://piao.qunar.com/ticket/list.htm?keyword=搜索地点&region=&from=mpl_search_suggest&page=页数`，这里既然搜索热门景点可用，那么就使用该关键词进行爬虫，只需要进行页码参数的调整即可。

## 1.2 分析页面元素

​		进入开发者面板后使用元素选择器可以看到在如图的class下便是页面中每条记录数据的显示文字了，例如下图中的”上海·上海·浦东新区“这个地址属性便在`<span class="area">`标签下显示：

![NM8eGq.jpg](https://s1.ax1x.com/2020/06/19/NM8eGq.jpg)

​		进一步的，我们可以在例如`<div class="intro color999">`等标签内识别到介绍等相关信息，经过分析，这里我们提取出来的信息包括了景点名称、级别、所在区域、起步价、销售量、热度、地址、经纬度、标语、详情网址。这里的经纬度在项目初期是计划使用百度API根据获取的地址信息进行获取，因此项目的展示需要百度API的支持。但后来无意发现，经纬度其实本网页html中自带，因此似乎白费百度API了。网页内自涵的经纬度如下所示：

![NM3zGt.jpg](https://s1.ax1x.com/2020/06/19/NM3zGt.jpg)

# 2 爬取数据

​		这里发现这个网站居然是基于http协议而且没有反爬机制的，因此直接采用request爬取即可，在爬取过程中我曾经因为网络不好被拒绝请求，一度以为是被反爬关了小黑屋，就在我要用scrapy解决反爬问题的时候，我换了个网络环境，神奇的解决了这个棘手的问题。这里使用`xpath`进行内容的匹配，比正则好用许多。以下便是爬取内容的函数：

```python
def getList():
    place = '热门景点'
    url = 'http://piao.qunar.com/ticket/list.htm?keyword='+ place +'&region=&from=mpl_search_suggest&page={}'
    i = 1
    sightlist = []
    while i < 400:
        page = getPage(url.format(i)) #这里调用了getPage函数获取了网页数据
        selector = etree.HTML(page.text)
        print('正在爬取第', str(i), '页景点信息')
        i += 1
        informations = selector.xpath('//div[@class="result_list"]/div')
        for inf in informations: #获取必要信息
            sight_name = inf.xpath('./div/div/h3/a/text()')[0]
            sight_level = inf.xpath('.//span[@class="level"]/text()')
            if len(sight_level):
                sight_level = sight_level[0].replace('景区','')
            else:
                sight_level = 0
            sight_area = inf.xpath('.//span[@class="area"]/a/text()')[0]
            # print(sight_area)
            sight_hot = inf.xpath('.//span[@class="product_star_level"]//span/text()')[0].replace('热度 ','')
            sight_add = inf.xpath('.//p[@class="address color999"]/span/text()')[0]
            sight_add = re.sub('地址：|（.*?）|\(.*?\)|，.*?$|\/.*?$','',str(sight_add))

            sight_slogen = inf.xpath('.//div[@class="intro color999"]/text()')
            if len(sight_slogen):
                sight_slogen = inf.xpath('.//div[@class="intro color999"]/text()')[0]
            else:
                sight_slogen = "null"
            sight_price = inf.xpath('.//span[@class="sight_item_price"]/em/text()')
            if len(sight_price):
                sight_price = sight_price[0]
            else:
                i = 0
                break
            sight_soldnum = inf.xpath('.//span[@class="hot_num"]/text()')[0]
            sight_point = inf.xpath('./@data-point')[0]
            sight_url = inf.xpath('.//h3/a[@class="name"]/@href')[0]
            sightlist.append([sight_name,sight_level,sight_area,float(sight_price),int(sight_soldnum),float(sight_hot),sight_add.replace('地址：',''),sight_point,sight_slogen,sight_url])
        time.sleep(15)
    return sightlist
```

这里爬取的信息包括景点名称、级别、所在区域、起步价、销售量、热度、地址、经纬度、标语、详情网址。本项目实际并未完全使用，全部数据存储在csv文件中，再开发可用性高（实际上谁会去用呢？）

当然，有很多景点存在无销量、无星级、无介绍等情况，这里都在爬取的过程中使用了条件判断，如果为空就赋值为空或者0，这就是缺省值的处理。也可以引用上篇文章中的基于`sklearn`的缺省值处理方法，这里仿佛犯不着。

另一方面，地址的匹配使用`re.sub()`去除冗余信息，这里不赘述。这里使用了`pandas`库进行爬取数据的csv文件存储：

```python
def listToExcel(list,name):
    df = pd.DataFrame(list,columns=['景点名称','级别','所在区域','起步价','销售量','热度','地址','经纬度','标语','详情网址'])
    df.to_csv(name + ".csv", sep=',')
```

由于项目后期直接使用了爬取的经纬度，这里不需要借助API转换地址为经纬度，这里也给出这部分代码留作学习备份：

```python
def getBaiduGeo(sightlist,name):
    ak = '密钥'
    headers = {
    'User-Agent' :'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.113 Safari/537.36'
    }
    address = 地址
    url = 'http://api.map.baidu.com/geocoder/v2/?address=' + address  + '&output=json&ak=' + ak
    json_data = requests.get(url = url).json()
    json_geo = json_data['result']['location']
```

观察获取的json文件，location中的数据和百度api所需要的json格式基本是一样，还需要将景点销量加入到json文件中，最后将整理好的json文件输出到本地文件中。

```python
def getBaiduGeo(sightlist,name):
    ak = '密钥'
    headers = {
    'User-Agent' :'Mozilla/5.0 (Macintosh; Intel Mac OS X 10_12_6) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.113 Safari/537.36'
    }
    list = sightlist
    bjsonlist = []
    ejsonlist1 = []
    ejsonlist2 = []
    num = 1
    for l in list:
        try:
            try:
                try:
                    address = l[6]
                    url = 'http://api.map.baidu.com/geocoder/v2/?address=' + address  + '&output=json&ak=' + ak
                    json_data = requests.get(url = url).json()
                    json_geo = json_data['result']['location']
                except KeyError,e:
                    address = l[0]
                    url = 'http://api.map.baidu.com/geocoder/v2/?address=' + address  + '&output=json&ak=' + ak
                    json_data = requests.get(url = url).json()
                    json_geo = json_data['result']['location']
            except KeyError,e:
                    address = l[2]
                    url = 'http://api.map.baidu.com/geocoder/v2/?address=' + address  + '&output=json&ak=' + ak
                    json_data = requests.get(url = url).json()
                    json_geo = json_data['result']['location']
        except KeyError,e:
            continue
        json_geo['count'] = l[4]/100
        bjsonlist.append(json_geo)
        ejson1 = {l[0] : [json_geo['lng'],json_geo['lat']]}
        ejsonlist1 = dict(ejsonlist1,**ejson1)
        ejson2 = {'name' : l[0],'value' : l[4]/100}
        ejsonlist2.append(ejson2)
        print '正在生成第' + str(num) + '个景点的经纬度'
        num +=1
    bjsonlist =json.dumps(bjsonlist)
    ejsonlist1 = json.dumps(ejsonlist1,ensure_ascii=False)
    ejsonlist2 = json.dumps(ejsonlist2,ensure_ascii=False)
    with open('./points.json',"w") as f:
        f.write(bjsonlist)
    with open('./geoCoordMap.json',"w") as f:
        f.write(ejsonlist1)
    with open('./data.json',"w") as f:
        f.write(ejsonlist2)
```

本项目最终采用了以下方式输出json数据文件：

```python
def datatojson(sightlist):  #直接生成json数据
    bjsonlist = []
    ejsonlist1 = []
    ejsonlist2 = []
    num = 1
    for l in sightlist:
        json_geo = {}
        p = '(.*?),(.*?)$'
        geo = re.findall(p,l[7])[0]
        json_geo['lat'] = geo[1]
        json_geo['count'] = l[4]/100
        json_geo['lng'] = geo[0]
        bjsonlist.append(json_geo)
#        print('正在生成第', str(num), '个景点的经纬度')
        ejson1 = {l[0] : [geo[0],geo[1]]}
        ejsonlist1 = dict(ejsonlist1,**ejson1)
        ejson2 = {'name' : l[0],'value' : l[4]/100}
        ejsonlist2.append(ejson2)
        num +=1
    bjsonlist =json.dumps(bjsonlist)
    ejsonlist1 = json.dumps(ejsonlist1, ensure_ascii=False)
    ejsonlist2 = json.dumps(ejsonlist2, ensure_ascii=False)
    with open('./points.json',"w") as f:
        f.write(bjsonlist)
    with open('./geoCoordMap.json',"w", encoding='utf-8') as f:
        f.write(ejsonlist1)
    with open('./data.json',"w", encoding='utf-8') as f:
        f.write(ejsonlist2)
```

​		这里生成的三个json文件，一个是给百度地图api引入用的，另俩个是给echartsMap引入用的。

# 3 数据可视化

## 3.1 热力图生成

​		将前文所述的百度地图api示例中的源代码复制到解释器中，添加密钥（需要自己申请），保存为html文件，打开就可以看到和官网上一样的显示效果。echarts需要在实例页面，点击页面右上角的EN切换到英文版，然后点击download demo下载完整源代码。

​		根据[html导入json文件](http://www.jb51.net/article/36678.htm)修改网页源码，导入json文件。

```html
#百度地图api示例代码中各位置修改部分
<head>
    <script src="http://libs.baidu.com/jquery/2.0.0/jquery.js"></script>
</head>
<script type="text/javascript">
    $.getJSON("points.json", function(data){
        var points = data;
        script中原有函数；
        });
</script>       
```

​		此时直接打开baiduMap.html或者echartMap.html均不会正常显示，F12后得知服务器异常。因此这里需要创建一个服务器才能在本地显示（这两个只能运行于服务器），这里进入需要打开的html文件目录，首先将需要打开的html文件重命名为`index.html`，然后在此目录下调用控制台。python3环境下需使用`python -m  http.server`开启本地服务器，收取到成功开启的提示后在浏览器中打开[http://127.0.0.1:8000/](http://127.0.0.1:8000/)，即可浏览index.html。在控制台中Ctrl+C关闭本地服务器。

​	baiduMap.html以及echartMap.html的预览效果依次如下所示：

![NM6i9K.jpg](https://s1.ax1x.com/2020/06/19/NM6i9K.jpg)

![NM6F1O.jpg](https://s1.ax1x.com/2020/06/19/NM6F1O.jpg)

​	这是爬取了400页，每页15条数据后的结果，因此有些许简陋。进一步的数据完善下次一定，下次一定！

## 3.2 数据进一步可视化

​		在此基础上，为了进一步分析数据，进一步针对一些特性绘制Echarts：

### 3.2.1 绘制热门景点前20的柱状图

```python
# 绘制热门景点前20的柱状echart
def first_20_echart():
    # 设置行名
    columns = name_list
    # 设置数据
    data1 = top_list
    # 设置柱状图的主标题与副标题
    bar = pyecharts.Bar(title="热门景点前20的柱状图")

    bar.add("热度", columns, data1, mark_line=["average"],xaxis_rotate=30,  mark_point=["max", "min"])
    # 生成本地文件（默认为.html文件）
    bar.render('first_20_echart.html')
```

![NMytfK.jpg](https://s1.ax1x.com/2020/06/19/NMytfK.jpg)

## 3.2.2 绘制主要城市景点数漏斗图

```python
# 绘制主要城市热门景点数漏斗图
def nation_hotspot_echart():

    fl = pyecharts.Funnel("主要城市热门景点数", "2020年6月18日 22:36:24", title_pos='left', width=1400, height=700)
    fl.add("景点数", add_key, add_value, is_label_show=True, label_formatter='{b}{c}', label_pos='outside')
    fl.render('nation_hotspot_echart.html')
```

![NMyJFx.jpg](https://s1.ax1x.com/2020/06/19/NMyJFx.jpg)

## 3.2.3 绘制top20城市的3A、4A、5A景区数

```python
# 绘制top20城市的3A、4A、5A景区数堆叠柱形图
def nA_spot_echart():
    # 设置行名
    columns = add_key
    # 设置数据
    data1 = level_3a
    data2 = level_4a
    data3 = level_5a
    # 设置柱状图的主标题与副标题
    bar = pyecharts.Bar(title="top20城市的3A、4A、5A景区数")

    bar.add("3A景区数", columns, data1, mark_line=["average"], xaxis_rotate=30, mark_point=["max", "min"], is_stack=True)
    bar.add("4A景区数", columns, data2, mark_line=["average"], xaxis_rotate=30, mark_point=["max", "min"], is_stack=True)
    bar.add("5A景区数", columns, data3, mark_line=["average"], xaxis_rotate=30, mark_point=["max", "min"], is_stack=True)
    # 生成本地文件（默认为.html文件）
    bar.render('nA_spot_echart.html')
```

![NMyBmd.jpg](https://s1.ax1x.com/2020/06/19/NMyBmd.jpg)

### 3.2.4 绘制哪里去得起极坐标图

这里采用了与前面热力图导入Echart一样的方式，利用html导入：

```html
<!DOCTYPE html>
<html style="height: 100%">
   <head>
       <meta charset="utf-8">
   </head>
   <body style="height: 100%; margin: 0">
       <div id="container" style="height: 100%"></div>
       <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/echarts/dist/echarts.min.js"></script>
       <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/echarts-gl/dist/echarts-gl.min.js"></script>
       <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/echarts-stat/dist/ecStat.min.js"></script>
       <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/echarts/dist/extension/dataTool.min.js"></script>
       <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/echarts/map/js/china.js"></script>
       <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/echarts/map/js/world.js"></script>
       <script type="text/javascript" src="https://api.map.baidu.com/api?v=2.0&ak=9hmPUqXOhGLfg8xyqzWQ9C21jU7ILXxR"></script>
       <script type="text/javascript" src="https://cdn.jsdelivr.net/npm/echarts/dist/extension/bmap.min.js"></script>
       <script type="text/javascript">
var dom = document.getElementById("container");
var myChart = echarts.init(dom);
var app = {};
option = null;
var data = [
    [30.0, 499.0, 156.23], 
	[55.0, 388.0, 147.99], 
	[3.5, 338.0, 106.75], 
	[20.0, 308.0, 75.93], 
	[20.0, 298.0, 33.88], 
	[0.3, 298.0, 84.17], 
	[30.0, 268.0, 117.67], 
	[72.0, 258.0, 165.0], 
	[72.0, 244.9, 179.79], 
	[37.3, 225.0, 102.16], 
	[28.0, 210.0, 110.83], 
	[69.0, 208.0, 138.5], 
	[63.8, 199.0, 140.93],
	[0.6, 188.0, 94.3], 
	[91.9, 185.0, 135.97], 
	[0.8, 170.0, 92.67], 
	[45.0, 168.0, 80.25], 
	[135.0, 160.0, 147.5], 
	[60.0, 160.0, 108.82], 
	[10.0, 140.0, 58.5], 
	[39.0, 138.0, 91.67], 
	[5.0, 90.0, 69.48], 
	[1.9, 89.0, 73.34], 
	[52.0, 82.8, 62.27]
];
var cities = ['上海', '湖北', '海南', '四川', '云南', '江苏', '陕西', '甘肃', '湖南', '广东', '重庆', '西藏', '天津', '青海', '辽宁', '浙江', '江西', '山西', '山东', '北京', '广西', '贵州', '河南', '安徽'];
var barHeight = 30;

option = {
    title: {
        text: '到底哪些地方去得起？？',
        subtext: '钱包允许（数据来源：爬取的去哪旅行）'
    },
    legend: {
        show: true,
        data: ['价格范围', '均值']
    },
    grid: {
        top: 100
    },
    angleAxis: {
        type: 'category',
        data: cities
    },
    tooltip: {
        show: true,
        formatter: function (params) {
            var id = params.dataIndex;
            return cities[id] + '<br>最低：' + data[id][0] + '<br>最高：' + data[id][1] + '<br>平均：' + data[id][2];
        }
    },
    radiusAxis: {
    },
    polar: {
    },
    series: [{
        type: 'bar',
        itemStyle: {
            color: 'transparent'
        },
        data: data.map(function (d) {
            return d[0];
        }),
        coordinateSystem: 'polar',
        stack: '最大最小值',
        silent: true
    }, {
        type: 'bar',
        data: data.map(function (d) {
            return d[1] - d[0];
        }),
        coordinateSystem: 'polar',
        name: '价格范围',
        stack: '最大最小值'
    }, {
        type: 'bar',
        itemStyle: {
            color: 'transparent'
        },
        data: data.map(function (d) {
            return d[2] - barHeight;
        }),
        coordinateSystem: 'polar',
        stack: '均值',
        silent: true,
        z: 10
    }, {
        type: 'bar',
        data: data.map(function (d) {
            return barHeight * 2;
        }),
        coordinateSystem: 'polar',
        name: '均值',
        stack: '均值',
        barGap: '-100%',
        z: 10
    }]
};
;
if (option && typeof option === "object") {
    myChart.setOption(option, true);
}
       </script>
   </body>
</html>
```

<img src="https://s1.ax1x.com/2020/06/19/NMchon.jpg" alt="NMchon.jpg" style="zoom:76%;" />

# 源代码

[shimmerjordan的gayhub](https://github.com/shimmerjordan/tourSpotList)

# 这次一定！

关于下次一定的事情，这里我终于没有放鸽子，电脑没关爬了一下，4000多页的数据全部爬取，所生成的热力图如下所示：

![N3V8RU.jpg](https://s1.ax1x.com/2020/06/21/N3V8RU.jpg)

![N3V3GT.jpg](https://s1.ax1x.com/2020/06/21/N3V3GT.jpg)
