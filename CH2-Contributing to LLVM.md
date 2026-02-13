- Contributing to an open source project can take many forms. Here are a few examples that we will cover in this chapter:
  - Reporting issues
  - Engaging in conversations in the public forum
  - Reviewing code
  - Contributing patches
- Include the configuration:
  - The version of LLVM
  - The OS
  - How LLVM was built
  - Anything that you think may be relevant (e.g., host CPU, etc.)
  - Upload the reduced input
  - Provide the steps to reproduce
    - Download myCoupleTensOfLines.ll
    - Run opt -passes=YY myCoupleTensOfLines.ll
    - Measure the compile time
    - Download myCoupleTensOfLinesWithASmallTweak.ll
    - Run opt -passes=YY myCoupleTensOfLinesWithASmallTweak.ll
    - Measure the compile time
    - Notice how the small tweak increases the compile time by 10x
  - Explain how the observed behavior departs from the expected behavior
  - Add labels to help identify which component is at fault if you can

- 要点：
  ```
  The takeaway is the more effort you spend on your report, the more likely it is that it can be fixed quickly. Another way to see it is that all the effort that you put in is someone else’s saved effort, and given that you have more context, you will be more effective at filling in the initial blanks.
  ```

- 请不要为那些修复成本极低的问题提交 issue——写报告的时间可能比修 bug 还长，这类问题只会无谓地挤占 bug 跟踪系统的资源。
  - The idea here is to avoid flooding the bug tracker with things that take more time to file than fixing them.


#### 附件
- LLVM Community Code of Conduct: https://llvm.org/docs/CodeOfConduct.html
- 即时沟通：https://discord.com/channels/636084430946959380/642374147640524831
- LLVM线上论坛：https://discourse.llvm.org/c/announce/46
