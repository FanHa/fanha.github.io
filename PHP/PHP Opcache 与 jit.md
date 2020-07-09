+ Opcache 将执行文件解析成的Opcode 缓存起来,下次调用时省掉 "代码->OPcode"的过程, 直接把缓存的OPCode供Zend 引擎执行,这里Zend引擎依然有很多戏份;
+ jit 想更进一步,直接缓存机器可以执行的机器码,减少Zend引擎的戏份