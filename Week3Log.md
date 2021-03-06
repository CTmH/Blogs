# 第三周日志
## 主要工作
1. 完成最短路径函数并改造
2. 完成站点遍历函数
3. 完成根据线路名称检索站点的函数
4. 完成根据站点名称检索同线路站点的函数
5. 完成线路输出函数

## 主要问题和解决方法
### 如何保存和输出查询结果
根据根据项目需求分析，我们发现所有的输出都是一组站点序列，因此我们设定数据结构path，用于保存和输出查询结果，path有两个变量：
变长数组用于保存站点对应的序变成数字用于保存站点对应的序号，另一个变量保存站点长度。变成数字当中保存的序号序号，而非站点结构体，这样可以减少存储空间的佔用，同时提高数据变成数字当中保存的是序号序号，而非站点结构T这样可以减少存储空间的佔用，
同时提高数据传输的效率。在函数间传递的效率。
减少了输出函数的种类。

这样不仅仅可用于最短路径和站点遍历路径的查询，线路查询结果也可以用这个结构体来表示，减少了输出函数的种类。

### 如何设计站点遍历函数
在概要设计阶段，我们一度以为站点遍历是就是简单的题TSP问题，即查找一条最短路径经过图上的每一个点仅一次。但是我们在详细设计阶段发现这个问题并非是单纯的TSP问题，因为在TSP问题当中同一个站点最多被访问一次，而地铁站的遍历问题同一个站点可以被多次访问。
我们通过检索最终找到了一篇相关的论文，论文描述的解决这个问题的方法：

首先根据地铁线路图构建地铁站之间的最短通路图，最短通路图当中的点就是地铁线路图当中的地铁站，边是两点之间的最短线路的边权和。在最短通路图上的TSP问题的解就是线路图中待求问题的解。

这一算法的正确性可以在论文[《具有重复路径的有向TSP问题》](https://kns.cnki.net/KCMS/detail/detail.aspx?dbcode=CJFQ&dbname=CJFD2010&filename=CAIZ201017222&v=MjEwNjhIOUhOcUkxSFpvUjhlWDFMdXhZUzdEaDFUM3FUcldNMUZyQ1VSN3FmWnVkbUZ5N25XcnZQSml6Q2RMRzQ=)当中找到证明。

具体的实现代码如下：

我们首先在系统类初始化时生成地铁线路图的最短线路图。

```cpp
void SearchSys::get_all_pairs_shorest_graph()
{
  VertexMap v_origin_map = get(vertex_index, mtgph); //地铁线路图的点集
  int v_size = num_vertices(mtgph);
  vector<vector<int>> D(v_size, vector<int>(v_size, 0)); //线路图的最小代价矩阵

  johnson_all_pairs_shortest_paths(mtgph, D); //计算线路图的最小代价矩阵

  for (int i = 0; i < v_size; i++)
    for (int j = 0; j < v_size; j++)
      if (D[i][j] != (std::numeric_limits<int>::max)() && i != j)
        {
          //转换为最短通路图
          add_edge(v_origin_map[i], v_origin_map[j], D[i][j], all_pairs_shorest_graph);
        }
  return;
}
```

再设计函数，查找最短线路图上输入起点的TSP解的路径序列，然后获得序列上相邻点之间的最短路径，
将这些路径拼接就构成了所求解。
```cpp
//获得在最短通路图上的TSP路径
  metric_tsp_approx_from_vertex(all_pairs_shorest_graph, src,
                                get(edge_weight, all_pairs_shorest_graph), get(vertex_index, all_pairs_shorest_graph),
                                tsp_tour_visitor<back_insert_iterator<vector<Vertex> > >
                                (back_inserter(travel_trail)));
  Path travel_path, tmp;

  int last = -1, tmp_last = -1;
  for (vector<Vertex>::iterator itr = travel_trail.begin(); itr != travel_trail.end(); ++itr)
    {
      if (graph_station_list[*itr].id != last)
        {

          if (last != -1)
            {
              //获得两个TSP路径点之间的实际路径
              tmp = find_spath(graph_station_list[last].id, graph_station_list[*itr].id);
              
              vector<int>::iterator itmp = tmp.stnid.begin();
              //拼接路径
              for (; itmp != tmp.stnid.end(); ++itmp)
                {
                  if(graph_station_list[*itmp].id != tmp_last)
                    travel_path.stnid.push_back(*itmp);
                  tmp_last = graph_station_list[*itmp].id;
                }
            }
          else travel_path.stnid.push_back(graph_station_list[*itr].id);
        }
      last = graph_station_list[*itr].id;
    }
```

### 选择合适的生成最短线路图函数
求解全源最短路径问题的常见方法是Floyd-Warshall算法，时间复杂度为O(n^3)。但是地铁线路图的特点是图的边相对较少，因此我们查到了Johnson算法，
该算法时间复杂度为O(nVlogV)，更适合边少点多的地铁线路图。

### 线路查询
由于在读取线路图数据时已经生成了线路的站点表，因此只需获得站点名对应的索引，就可以直接取得线路站点。
```cpp
  map<string, int>::iterator l_it;
  l_it = Line_nameToNum.find(linename); //查询索引
  …
  line.stnid = Line_list[line_id]; //取得线路
  line.len = Line_list[line_id].size();
  …
```
**至此所有的查询功能已经实现**

### 图形用户界面

#### 处理从xml文件的输入

使用xml.etree.cElementTree模块进行xml文件的信息处理操作，获得用于生成线路图的站点线路信息
```python
tree = ET.parse(path)
    map = tree.getroot()
    # 长宽记录
    w = int(map.find('w').text)
    h = int(map.find('h').text)
     for line in map.findall('line'):  # 查找所有线路
     ......
     for station in line: # 查找线路下的所有站点
     ......
```
#### 主要组件

GUI包括Lable、输入框、按钮，以及地铁线路图、输出消息栏。
使用tkinter组件建立窗口、按钮等组件

使用标签创建地铁线路图，每个标签都代表一个站点
```python
for tmp_station in station_list:
    text = tmp_station.sname
    mapx = tmp_station.mapx
    mapy = tmp_station.mapy
    lname_list = tmp_station.lname_list
    lname = lname_list[0]
    color = "white"
    l = tk.Label(window, text=text, bg= color, font=('Aerial', 8))//根据temp_station的信息初始化标签
    l.place(x=mapx, y=mapy)
    station_label_list.append((tmp_station, l))
```
#### 调用查询程序
利用subprocess调用可执行文件Metroplan
```python
 child = subprocess.Popen(['./Metroplan', 'BEIJING','-c', cost, '-s', start_name, end_name]) 
    child.wait()  # 子进程等待
```
返回结果采用共享文件，之后根据结果输出图形变换
```python
 tmp_sname_list = []
    input_path = 'shortest_path.txt' # 通过记录的结果文件，读取对应文件
    input_file = codecs.open(input_path, 'r')
……
# 根据后端返回的信息，按顺序在地图上显示出来，并使用红色表示经历过的，蓝色表示正在经历的
def show_line(tmp_sname_list):
    num = 0
    for tmp_sname in tmp_sname_list:
        num  = num +1  # 每次高亮显示一个站点，记录+1
        NUM["text"] = "经过的站点数："+str(num)
        for (tmp_station, l) in station_label_list:
            if tmp_sname == tmp_station.sname:
                l['bg'] = 'blue'
                # last_one = i
                l.update()
                time.sleep(.200)
                l['bg'] = 'red'
                # ls.update()
                break
```
