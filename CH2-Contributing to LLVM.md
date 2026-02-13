- Contributing to an open source project can take many forms. Here are a few examples that we will cover in this chapter:
  - Reporting issues
  - Engaging in conversations in the public forum
  - Reviewing code
  - Contributing patches

- Remember that you do not have to start with hard-to-fix problems. You can start by fixing small issues or typos (like good first issue)! Whatever helps get you started with the process is a good learning experience.

- Issue Included the configuration:
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
- You understood that the open source community cannot survive on patch contributions alone.
- 提交补丁前：
  - How would you have implemented this patch?
  - Does the contribution make sense?
  - Is there a better way to do this?
  - Is this even necessary? Maybe something already solves that problem elsewhere.
  - Learn the surrounding code.
  - Understand how things connect together.

- 请不要为那些修复成本极低的问题提交 issue——写报告的时间可能比修 bug 还长，这类问题只会无谓地挤占 bug 跟踪系统的资源。
  - The idea here is to avoid flooding the bug tracker with things that take more time to file than fixing them.
- 遇到问题或者想请教问题，通过discord请教那些专家，他们会很乐意
- 通过命令找到制定文件或者目录的修改者
  ```
  git log --pretty=format:"%aN<%ae> (%as)-%h %s" -- llvm/lib/Transforms/IPO/AlwaysInliner.cpp
  Kazu Hirata<kazu@google.com> (2025-02-18)-5d4eb08379cc [Analysis] Remove skipSCC (#127412)
  Nikita Popov<npopov@redhat.com> (2024-12-12)-2be41e7aee1c [AlwaysInline] Fix analysis invalidation (#119566)
  Nikita Popov<npopov@redhat.com> (2024-11-27)-43ee6f7a01fc [AlwaysInline] Avoid unnecessary BFI fetches (#117750)
  ```
- 提交补丁后如何找到对应reviewer ？
- 补丁建议
  - 提交补丁原则：尽量准确详细，别人复现复习效果越高
  - 尽量循序渐进，大的修改需要和社区先沟通
  - 增加测试用例： Obviously, the test needs to fail without your change, otherwise you are not testing the right thing.
  - approved后commit被压缩
  - 跟踪pr，一周后无人回应则ping reviewer或者上discord通知
  - 合入后，查看buildbot是否运行ok，否则revert

#### 附件
- LLVM Community Code of Conduct: https://llvm.org/docs/CodeOfConduct.html
- 即时沟通：https://discord.com/channels/636084430946959380/642374147640524831
- LLVM线上论坛：https://discourse.llvm.org/c/announce/46
- Code review: https://llvm.org/docs/CodeReview.html
- coding standard: https://llvm.org/docs/CodingStandards.html
- buildbot：https://lab.llvm.org/buildbot/#/console
