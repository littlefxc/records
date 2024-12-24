---
status: Done
Tags:
  - excel
---

# Excel合并单元格填充序列的方法

1. 步骤1：=COUNTA($A$2:A2)+1  
2. 步骤2：ctrl + enter  

# Excel 截取指定字符之前的字符串

![[Excel截取指定字符之前的字符串.png]]

例如： A2 单元格的内容是"设备维修K20180910"，要截取'K'之前的字符串  
1. 步骤1：=left(A2,find('K',A2,1)-1)  
2. 步骤2：enter  

# Excel生成不重复的UUID

=CONCATENATE(DEC2HEX(RANDBETWEEN(0,4294967295),8),DEC2HEX(RANDBETWEEN(0,42949),4),,DEC2HEX(RANDBETWEEN(0,42949),4),DEC2HEX(RANDBETWEEN(0,42949),4),DEC2HEX(RANDBETWEEN(0,4294967295),8),DEC2HEX(RANDBETWEEN(0,42949),4))