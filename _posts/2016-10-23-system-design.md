### google doc

要点:

1. multi user
2. history 的存储
3. revert

参考

[so](http://stackoverflow.com/questions/5772879/how-do-you-write-a-real-time-webbased-collaboration-tool-such-as-google-docs)
[quora](https://www.quora.com/How-is-collaborative-document-editing-implemented-in-Google-Docs)

**git diff 原理**

From a longest common subsequence it's only a small step to get diff-like output: if an item is absent in the subsequence but present in the original, it must have been deleted. (The '–' marks, below.) If it is absent in the subsequence but present in the second sequence, it must have been added in. (The '+' marks.)

[](http://www.mathertel.de/Diff/DiffDoku.aspx)

仅使用 git diff （不使用参数）会只显示工作目录中已添加到 index 的更改
