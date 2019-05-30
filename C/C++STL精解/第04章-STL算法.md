### 第04章-STL算法

#### 4.2 非修改性算法

##### 4.2.2 元素计数算法

```c++
vector<int> nums {1,1,2,3,4,5};
int cnt = count(nums.beign(),nums.end(), 1); //cnt=2
cnt = count(nums.begin(), nums.end(), bind2nd(greater<int>(),2))//3
```

##### 4.2.3 最小值和最大值算法

```c++
bool AbsLess(int a, int b) {
  return abs(a)<abs(b);
}
auto min = min_element(nums.begin(), nums.end(), AbsLess);//*min为1，返回的min为指向该元素的迭代器
auto max = max_element(nums.begin(), nums.end(), AbsLess);//*max为5	
```

##### 4.2.4 搜索算法

```c++
auto pos = find(nums.begin(), nums.end(), 2);
cout<<distance(nums.begin(), pos)<<endl;//返回元素2的索引，即2
pos = find_if(nums.begin(), nums.end(), bind2nd(greater<int>(),3));//返回第一个大于3的元素的位置
auto pos1 = search_n(nums.begin(), nums.end(),3,1);//即3个连续等于1的元素的起始位置，这里找不到，返回的是nums.end()，最后的1也可以用仿函数
//连续两个相等的元素
auto it = adjacent_find(nums.begin(),nums.end());//返回nums.begin()，即1,1。也可以加一个仿函数作为相等的判断条件
```

##### 4.2.5 比较算法

- equal实现两个容器对象的比较
- mismatch用于寻找两个容器对象之间两两相异的对应元素
- lexicographical_mismatch：一一比较两容器中的元素

#### 4.3 修改性算法

##### 4.3.1 复制

```c++
copy(nums.begin(), nums.end(), other.begin()+1);//将nums中的元素复制到other中的begin()+1位置
copy_backward()
```

##### 4.3.2 转换

back_inserter、inserter、front_inserter

```c++
transform(nums.begin(), nums.end(), back_inserter(other));//将nums中元素以other.push_back的方式转入到other中
transform(nums.begin(), nums.end(), back_inserter(other),negate<int>());//仿函数形式，将所有nums中的元素乘以-1，然后push_back到other中
```

##### 4.3.3 互换

```c++
swap(nums, other);//交换两者的所有元素
swap_ranges(nums.begin(), nums.end(), other.begin(), other.end());//交换两者指定范围内的元素
```

##### 4.3.4 赋值

```c++
fill(nums.begin(), nums.end(), 1);//修改所有元素为1	
fill_n(nums.begin(), 2, 3);//修改前2个元素为3
generate(nums.begin(), nums.end(), gen);//gen为仿函数，无参，用来生成元素
generate_n();//类似fill_n，由仿函数gen生成元素
```

##### 4.3.5 替换

```c++
replace(nums.begin(), nums.end(), 5, 15);//将nums中所有5替换为15，其中5也可以通过仿函数来替代
```

##### 4.3.6 逆转

```c++
reverse(nums.begin(), nums.end());
reverse_copy(nums.begin(), nums.begin()+2, other.begin()+2);//将nums中的指定元素倒叙复制到other的指定位置后
```

##### 4.3.7 旋转

```c++
rotate(nums.begin(), nums.begin()+5, nums.end());//将nums.begin()+5元素移动到nums.begin旋转移动到nums.begin()位置
```

##### 4.3.8 排列

```c++
next_permutation(nums.begin(), nums.end());//升序排列
random_shuffle(nums.begin(), nums.end());//重排
partition(nums.begin(), nums.end(), pred);//根据仿函数将nums分成两块
stable_partition(nums.begin(), nums.end(), pred);//类似partition，但是会保持原容器中元素的相对顺序不变
```

#### 4.4 排序及相关操作算法

- sort和stable_sort支持对容器中所有元素的排序，但是需要访问随机迭代器，只适用于vector和deque型容器
- list通过成员函数sort排序
- 局部排序partial_sort
- nth_element：是所有位置在n之前的元素都小于等于它，所有位置在n之后的元素都大于等于它，有点类似partition
- merge合并两个容器中的元素、set_union两个元素中的并集、set_intersection两个元素的交集、set_difference两个容器的差集
- binary_search二分搜索法
- lower_bound返回第一个大于等于value的值、equal_range返回是lower_bound和upper_bound的共同返回值

#### 4.5 删除算法

```c++
remove(nums.begin(), nums.end(), 2);//移除nums中所有=2的元素
remove_if(nums.begin(), nums.end(), pred);//根据pred仿函数确定要移除的元素
//remove_copy和remove_copy_if用于在复制中删除元素
//unique移除容器中的重复元素，返回值为修正后的容器中的最有一个元素的位置
```







