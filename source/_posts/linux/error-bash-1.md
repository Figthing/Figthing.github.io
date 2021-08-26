title: 'shell /bin/bash^M: bad interpreter错误解决'
author: Figthing
tags:
  - linux
  - shell
categories:
  - linux
date: 2018-01-19 17:16:00
---
> 错误原因之一很有可能是你的脚本文件是DOS格式的, 即每一行的行尾以\r\n来标识, 其ASCII码分别是0x0D, 0x0A.可以有很多种办法看这个文件是DOS格式的还是UNIX格式的, 还是MAC格式的。

1. `vi filename`然后用命令`:set ff?`可以看到dos或unix的字样. 如果的确是dos格式的, 那么你可以用set ff=unix把它强制为unix格式的, 然后存盘退出. 再运行一遍看.

2. 用`joe filename`如果是DOS格式的, 那么行尾会有很多绿色的^M字样出现. 你也可以用上述办法把它转为UNIX格式的.

3. 用`od -t x1 filename`如果你看到有0d 0a 这样的字符, 那么它是dos格式的, 如果只有0a而没有0d, 那么它是UNIX格式的, 同样可以用上述方法把它转为UNIX格式的. 

> 转换不同平台的文本文件格式可以用

1. unix2dos或dos2unix这两个小程序来做. 很简单. 在djgpp中这两个程序的名字叫dtou和utod, u代表unix, d代表dos
2. 也可以用sed 这样的工具来做:    

 ```shell
 $ sed 's/^M//' filename > tmp_filename
 $ mv -f tmp_filename filename
 ```
  来做说明:^M并不是按键shift + 6产生的^和字母M, 它是一个字符, 其ASCII是0x0D, 生成它的办法是先按CTRL+V, 然后再回车(或CTRL+M)