## 1 简单数组
### 1.1 创建数组
数组中可以存放多个值。Bash Shell 只支持一维数组，初始化时不需要定义数组大小，与大部分编程语言类似，数组元素的下标由 0 开始。
```shell
#!/bin/bash
# 创建数组方式一
my_array=(A B "C" D)

# 创建数组方式二
array_name[0]=value0
array_name[1]=value1
array_name[2]=value2

# 读取数组内容
echo ${arr[0]}
```
### 1.2 数组遍历
#### 1.2.1 直接遍历元素
```sh
#!/bin/bash

arr=(1 2 3)

for v in "${arr[@]}"; do
	echo "$v"
done
```
#### 1.2.2 通过下标遍历
```sh
#!/bin/bash
arr=(1 2 3)
len="${#arr[@]}"

for ((i=0; i < $len; i++)); do
	echo "${arr[$i]}"
done
```
## 2 关联数组
Bash 支持关联数组，可以使用任意的字符串、或者整数作为下标来访问数组元素，类似 Map 结构。
```sh
# 创建方式一
declare -A site=(["google"]="www.google.com" ["runoob"]="www.runoob.com" ["taobao"]="www.taobao.com")

# 创建方式二
declare -A site
site["google"]="www.google.com"
site["runoob"]="www.runoob.com"
site["taobao"]="www.taobao.com"

# 访问方式
echo ${site["runoob"]}
```
## 3 全量获取方式
### 3.1 获取数组中的所有元素
```sh
#!/bin/bash

my_array[0]=A
my_array[1]=B
my_array[2]=C
my_array[3]=D

echo "数组的元素为: ${my_array[*]}"
echo "数组的元素为: ${my_array[@]}"
```
### 3.2 获取数组中的所有键
```sh
declare -A site
site["google"]="www.google.com"
site["runoob"]="www.runoob.com"
site["taobao"]="www.taobao.com"

echo "数组的键为: ${!site[*]}"
echo "数组的键为: ${!site[@]}"
```
### 3.3 获取数组的长度
通过 `my_array[*]` 获取数组的值列表，再通过 `#` 来获取数组长度
```sh
#!/bin/bash
# author:菜鸟教程
# url:www.runoob.com

my_array[0]=A
my_array[1]=B
my_array[2]=C
my_array[3]=D

echo "数组元素个数为: ${#my_array[*]}"
echo "数组元素个数为: ${#my_array[@]}"
```