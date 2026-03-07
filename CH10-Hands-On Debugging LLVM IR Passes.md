- print-before-all/print-after-all: Print the IR before (or after, respectively) each pass in the current pipeline.
- print-before/print-after=PassName1[,PassName2]*: Print the IR before (or after, respectively) PassName1, PassName2, and so on, where PassNameI is the CLI option name of the related pass (see Chapter 5 to find out how to find this name). Like all the CLI options that accept comma-separated lists in the LLVM CLI, you can specify this option several times. This is equivalent to providing the concatenated comma-separated list to one instance of the option
- to start producing debug logs in your pass, you must do the following:
  - 1. Include the Debug.h, from the Support library, in your pass.
  - 2. Define the DEBUG_TYPE macro. We recommend sticking to the same name as the one you used for the CLI option name unless it makes sense to group your debug log with other passes.
  - 3. Use the LLVM_DEBUG macro to start performing debug-log-specific tasks such as printing some message or the content of a variable

- The capabilities of llvm-extract are built around the ExtractGVPass and BlockExtractorPass passes from the IPO library. Generally speaking, if you want to extract a region from some IR, this functionality is available in the TransformsUtils library under the CodeExtractor class. This class can be used in the PartialInliner or the LoopExtractor passes.For instance, let’s assume we want to extract the foo and bar functions from some input IR. The following command does this for us:
  ```
  llvm-extract -S -func=bar -func=foo -o - input.ll
  ```
- To extract bb2 from bar, you can run the following command:
  ```
  $ llvm-extract -S -o - input.ll -bb=bar:bb2
  ```
 
