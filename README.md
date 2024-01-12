# SJTU CS2612 程序语言与编译原理课程大作业：寄存器分配

## 项目简介
本次项目作业利用课上所讲授的liveness分析方法,在一段程序已经做好基本快生成的情况下，进行寄存器分配，具体实现的功能有：  
* liveness分析
* 寄存器分配
* 汇编代码生成
* 在考虑系统调用的函数情况下对caller与callee saved寄存器进行了处理

## 如何使用
* 主程序为```backend_complier.cpp```，待分配程序以```output.txt```文档的形式给出，直接编译运行```backend_complier.cpp```即可对```output.txt```内的程序进行寄存器分配
* ```rg_cnt```维护了本次中寄存器可供分配的最大数量，修改```rg_cnt```的值可以改变寄存器分配的数量。（在实际使用时，考虑到实际的体系结构，我们允许的最大可使用寄存器数目是18.）

## 各部分具体功能简介
1. liveness分析：
   * 对每一条指令提取use与def集合
   * 根据整体程序之间的联系计算每一条指令中liveness的变量
2. 寄存器分配：
   * 完成基本操作：simplify，coalesce，freeze，spill
   * 遇到真spill情况对程序进行改写，引入将数据存取于内存的指令
3. 汇编代码生成：
   * 选取RISC-V作为汇编输出
   * 实现每一条指令的汇编生成
4. 函数调用中caller与callee saved寄存器考虑
   * 本项目中所有函数调用只考虑svc系统调用，因为在一个基本块中，倘若出现其余函数跳转，其理应作为一个新的基本块生成，而非以函数的形式出现在基本块中。而针对过程调用的参数传递，我们留出了x10-x17这8个参数寄存器用于传递参数，全程并不使用它们。
   * 我们将待分配寄存器的整个程序看作一个被调用函数，即在整个```output.txt```中，只有程序本身作为callee，需要考虑callee_saved寄存器问题，其余情况下的函数调用，我们以caller视角，仅考虑caller_saved寄存器问题。我们直接在汇编语句的层面上对此前得到的结果进行调整来完成过程调用的实现。
   * 我们最初的想法是简单的在汇编层面将所需备份的写入内存中，但曹钦翔老师认为这样对内存的占用较大，性能并不理想。曹钦翔老师对于此部分的建议是：引入临时变量对寄存器值进行保存，并对临时变量进行liveness分析，并且直接使用寄存器分配来完成。但这样的方式引入了额外的困难：原本的程序是不涉及寄存器的，仅有变量与变量之间的操作；按照老师的建议则会引入变量与寄存器之间的赋值，这会直接导致我们的算法无法正常工作；与此同时，我们原有的寄存器分配算法的正确性也将无法得到保证。对此，我们进行了分析，并得出可以直接在汇编层面上对代码进行调整就可以实现等价的操作。
   * 针对caller-saved寄存器的分配，我们采用的策略是：首先在不考虑过程调用的情况下完成上述寄存器分配算法。随后，对于涉及过程调用的指令，我们根据该指令的liveness情况分析出此时此刻寄存器的使用情况。只有既是caller-saved又在此时被占用的寄存器需要备份，在此时空闲的callee-saved寄存器中选取对应数目的寄存器完成备份即可；如果寄存器数目不足则写入内存。在完成过程调用后再从这些地方对备份的寄存器进行恢复。
   * 而针对callee-saved寄存器的分配，则是再caller-saved寄存器的基础上完成的。在程序的一开始我们应当对所有的callee-saved寄存器进行备份（无法得知调用者的使用情况），在程序结束后对其进行恢复。额外统计一个全局寄存器使用情况，对callee-saved寄存器只能选择全程没有被使用过的寄存器。如果空间不足则使用内存进行备份。
   * 本次项目的寄存器规划如下图所示：
     ![register_management](register_manage.jpg)


