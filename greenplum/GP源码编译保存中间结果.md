./configure ---

GP_INSTALL_PATH=/home/cao设置编译完成生成后安装文件目录

Makefile增加一行

CFALGS+=-save-temps （gcc编译保存中间结果)
make clean清空上次编译的结果
make执行编译过程(-j8利用所有cpu编译)
