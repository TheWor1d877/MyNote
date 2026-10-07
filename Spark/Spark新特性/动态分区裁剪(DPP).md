在SparkSQL运行之前会裁剪掉不用的列，这是静态裁剪
动态分区裁剪会根据静态裁剪的内容，自动过滤其他表中无用的数据
![[Attachments/Screenshot_20250831_105610.jpg]]