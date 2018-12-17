# 概要
diff 命令以逐行的方式，比较文本文件的异同处。如果要指定比较目录，则 diff 会比较目录中相同文件名的文件，但不会比较其中子目录。

# 语法
```
diff [OPTION]... FILES
```

## 选项
- `--normal`：  
输出正常差异。
- `-q, --brief`：  
仅当文件有差异时才报告。
- `-s, --report-identical-files`：  
当文件相同时报告。
- `-c, -C NUM, --context[=NUM]`：  



