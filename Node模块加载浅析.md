# Node模块加载浅析

Node中模块的加载顺序：核心模块 > 普通模块 > 自定义模块（文件模块）

Node中的模块寻址过程：

当前项目的node_modules -> 上一级的node_modules -> ...... -> node path路径的node_modules

若有使用相对路径、绝对路径引用的模块，则 

