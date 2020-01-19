# 第四周日志
## 主要工作
1. 实现基本的命令行交互界面
2. 进行单元测试
3. 进行集成测试
4. 进行性能分析
5. 进行代码质量分析
6. 进行内存分析
7. 根据分析结果改进软件
8. 实现图形用户交互界面
9. 完成项目统计和总结博客

## 主要问题和解决方法
1. 由于程序逻辑结构简单且大量调用外部类库，所以在单元测试时，主要采用语句覆盖和逻辑覆盖的白盒测试方法。
2. 代码质量分析采用cppcheck静态检测，遇到的主要错误是自增号后置，未使用的变量，过大或过小的变量作用域，结构体的值传递。
3. 内存测试采用 进行动态测试。此程序可以检测申请内存空间是否完全释放。经过检测，软件没有出现内存泄漏。
4. 性能测试采用perf，该软件检测运行过程中调用的函数，经过分析，发现主要性能消耗在调用最小生成树函数。这是由于在调用遍历函数生成线路时，要多次调用最小生成树函数。我们采用的优化方案是设立查找表，将已经生成的最小生成树储存起来，下次调用时就不需要重新计算。
5. 图形界面和主程序的交互方式采用读写文件的形式，图形界面调用查询程序，查询程序获得结果后写入文件，之后，图形界面程序读取文件内容，然后向用户展示结果。

### 白盒测试
选取程序中控制最复杂的函数main()

```cpp
  if(argc < 6 || argc > 7)//1 2
    {
      cout << "Wrong arg!" << endl;//3
      return "0";
    }
  string arg[7];//4
  for(int i = 0; i < argc; i++)
    arg[i] = argv[i];
  if(arg[2] != "-c")//5
    {
      cout << "Please set cost!"<< endl;//6
      return "0";
    }
  int cost = atoi(argv[3]);//7
  string op = arg[4];
  cout << "loading..." << endl;
  SearchSys ss(arg[1], cost);
  Path ans;
  if(op == "-s")//8
    {
      if(argc == 7)//9
        {
          try//10
            {
              ans = ss.Find_the_shrt_path(argv[5],argv[6]);
            }
          catch(char const* err)
            {
              cout << err << endl;
              return "0";
            }
          ss.print_path(ans);
          filenames = "shortest_path.txt";
          ss.save_path(ans, "shortest_path.txt");
        }
      else return "0";//11
    }
  else if(op == "-t")//12
    {
      try//13
        {
          ans = ss.Trave_metro(argv[5]);
        }
      catch(char const* err)
        {
          cout << err << endl;
          return "0";
        }
      ss.print_path(ans);
      filenames ="travel.txt";
      ss.save_path(ans, "travel.txt");
    }
  else if(op == "-l")//14
    {
      Path *pllist;//15
      int list_num = 0;
      try
        {
          list_num = ss.Find_line(argv[5], pllist);
        }
      catch(char const* err)
        {
          cout << err << endl;
          return "0";
        }
        filenames ="line.txt";
      ss.save_path(pllist, "line.txt", list_num);
      delete[] pllist;
    }
  else if(op == "-ln")//16
    {
      try//17
        {
          ans = ss.Find_line_with_name(argv[5]);
        }
      catch(char const* err)
        {
          cout << err << endl;
          return "0";
        }
      ss.print_path(ans);
        filenames = "line_with_num.txt";
      ss.save_path(ans, "line_with_num.txt");
    }
  else//18
      {
        cout<<"Command param error!"<<endl;
        return "0";
      }

  cout << "Path has been saved" <<endl;//19
  return filenames;
}
```

![白盒测试路径](./白盒测试.png "白盒测试路径图")

环形复杂度为9，设计9个线性独立路径：
- 1-3
- 1-2-3
- 1-2-4-5-6
- 1-2-4-5-7-8-9-10-19
- 1-2-4-5-7-8-9-11
- 1-2-4-5-7-8-12-13-19
- 1-2-4-5-7-8-12-14-15-19
- 1-2-4-5-7-8-12-14-16-17-19
- 1-2-4-5-7-8-12-14-16-18

对应的9个测试用例：

main -c
main -c -c -c -c -c -c -c -c -c
main BEIJING -s -s -s -s -s
main BEIJING -c 1 -s 苹果园 良乡大学城北
main BEIJING -c 1 -s 苹果园
main BEIJING -c 1 -t 海淀黄庄
main BEIJING -c 1 -l 苹果园
main BEIJING -c 1 -ln Line1
main BEIJING -c 1 -b a

**以上测试均通过

### 代码质量分析
使用cppcheck对代码进行分析后得到[代码质量分析报告](./检测报告.xml)
部分代表性结果如下：
```xml
        <error id="variableScope" severity="style" msg="The scope of the variable &apos;line_id&apos; can be reduced." verbose="The scope of the variable &apos;line_id&apos; can be reduced. Warning: Be careful when fixing this message, especially when there are inner loops. Here is an example where cppcheck will write that the scope for &apos;i&apos; can be reduced:\012void f(int x)\012{\012    int i = 0;\012    if (x) {\012        // it&apos;s safe to move &apos;int i = 0;&apos; here\012        for (int n = 0; n &lt; 10; ++n) {\012            // it is possible but not safe to move &apos;int i = 0;&apos; here\012            do_something(&amp;i);\012        }\012    }\012}\012When you see this message it is always safe to reduce the variable scope 1 level." cwe="398">
            <location file="/home/chez/文档/大学资料/大三上/软件工程/课程设计/metro-planning/Findline.cpp" line="20" column="15"/>
            <symbol>line_id</symbol>
        </error>
        <error id="unreadVariable" severity="style" msg="Variable &apos;line_id&apos; is assigned a value that is never used." verbose="Variable &apos;line_id&apos; is assigned a value that is never used." cwe="563">
            <location file="/home/chez/文档/大学资料/大三上/软件工程/课程设计/metro-planning/Findline.cpp" line="20" column="23"/>
            <symbol>line_id</symbol>
        </error>
        <error id="postfixOperator" severity="performance" msg="Prefer prefix ++/-- operators for non-primitive types." verbose="Prefix ++/-- operators should be preferred for non-primitive types. Pre-increment/decrement can be more efficient than post-increment/decrement. Post-increment/decrement usually involves keeping a copy of the previous value around and adds a little extra code." cwe="398">
            <location file="/home/chez/文档/大学资料/大三上/软件工程/课程设计/metro-planning/Findline.cpp" line="30" column="60"/>
        </error>
        <error id="noExplicitConstructor" severity="style" msg="Class &apos;record_predecessors&apos; has a constructor with 1 argument that is not explicit." verbose="Class &apos;record_predecessors&apos; has a constructor with 1 argument that is not explicit. Such constructors should in general be explicit for type safety reasons. Using the explicit keyword in the constructor means some mistakes when using the class can be avoided." cwe="398">
            <location file="/home/chez/文档/大学资料/大三上/软件工程/课程设计/metro-planning/Findpath.cpp" line="9" column="3"/>
            <symbol>record_predecessors</symbol>
        </error>
```
可见代码主要问题是自增号后置，未使用的变量，过大或过小的变量作用域，结构体的值传递，不当重名。

### 性能分析
