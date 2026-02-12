#### 章节目的
- Understand the different components that make a compiler
- Build and test the LLVM project
- Navigate LLVM’s directory structure and locate the implementation of different components
- Contribute to the LLVM project
<img width="1086" height="566" alt="image" src="https://github.com/user-attachments/assets/a649e34d-0a34-44d3-af78-1b43b81237df" />

- clang definition:  Clang, is a compiler driver(include frontend and ): it invokes the different tools in the right order and pulls the related dependencies from the standard library to produce the final executable code.
<img width="678" height="782" alt="image" src="https://github.com/user-attachments/assets/749da53d-b1a8-45c0-9861-61faa5e9d5b1" />

- In any case, the focus of this book is LLVM backends, so, why are we spending so much time on Clang?
  The reason is simple: Clang offers a familiar way to interact with LLVM constructs. By using the Clang frontend, you will be able to generate the LLVM intermediate representation (IR) by simply writing C/C++. We believe this is a gentler way to start your journey with LLVM backends.
