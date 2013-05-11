---
layout: post
title: "使用awk比较文件"
date: 2013-05-11 18:46
comments: true
categories: [bash, shell, linux, awk, diff]
---

平时在CLI环境里面需要经常比较两个小文件的内容，最近专门搜索了下，收集了两个简单的比较方案。

### 方案1：###

    cat a | grep -vFf b

如果想要不区分大小写，加-i参数即可

### 方案2：###

如果数据量到达1000w以上，grep很容易占满内存，无法使用

这个时候就需要使用awk了

比如要比较a和b两个文件； 
a             b
1             1
qw            2
qq            d
123           cd

列出b文件中完全不包含a文件的行

	awk 'ARGIND==1 {arr[$0]} ARGIND>1&&!($0 in arr) {print $0}' a b
   
	awk 'NR==FNR {arr[$0]} NR>FNR&&!($0 in arr) {print $0}' a b

解释：

首先awk会按顺序先处理a文件，在处理b文件；

然后根据awk的环境变量列表：

	$n      	 当前记录的第n个字段，字段间由FS分隔。
	$0           完整的输入记录。
	ARGC         命令行参数的数目。
	ARGIND       命令行中当前文件的位置(从0开始算)。
	ARGV         包含命令行参数的数组。
	CONVFMT      数字转换格式(默认值为%.6g)
	ENVIRON      环境变量关联数组。
	ERRNO        最后一个系统错误的描述。
	FIELDWIDTHS  字段宽度列表(用空格键分隔)。
	FILENAME     当前文件名。FNR同NR，但相对于当前文件。
	FS           字段分隔符(默认是任何空格)。
	IGNORECASE   如果为真，则进行忽略大小写的匹配。
	NF           当前记录中的字段数。
	NR           当前记录数。
	OFMT         数字的输出格式(默认值是%.6g)。
	OFS          输出字段分隔符(默认值是一个空格)。
	ORS          输出记录分隔符(默认值是一个换行符)。
	RLENGTH      由match函数所匹配的字符串的长度。
	RS           记录分隔符(默认是一个换行符)。 
	RSTART       由match函数所匹配的字符串的第一个位置。
	SUBSEP       数组下标分隔符(默认值是\034)。



NR==FNR时是在处理a，将当前行放入数组

NR>FNR时是在处理b，如果当前行不在数组中，则打印

相应的ARGIND==1 正在处理a，ARGIND > 1 正在处理文件b


