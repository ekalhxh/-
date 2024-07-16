# 在Windows上安装MSMPI并使用VScode运行MPI程序（2024.07.16）
## 1. 安装MSMPI。
## 2. 安装Visual Studio，安装时需要选择C++套件。
## 3. 安装Intel oneAPI开发套件
`base toolkit`和`hpc toolkit`这两个，后者自带了ifort。如果前一步没有安装C++套件，这一步安装时会提示。
## 4. 创建mpi.mod文件的储存目录。
找到MSMPI的include目录，默认为`C:\Program Files (x86)\Microsoft SDKs\MPI\Include`。在其中新建文件夹intel。将include目录中的`mpi.f90`以及其子目录x64中的`mpifptr.h`复制到intel目录中。
## 5. 编译mpi.f90。
以管理员身份打开`Intel oneAPI command prompt for IA32 for Visual Studio 2022`（不要用cmd，cmd调用ifort或ifx很麻烦，后面会讲）。cd到intel文件夹，输入`ifort -c mpi.f90`编译得到mpi.mod等文件。
## 6. 创建想要运行代码所在的文件夹。
在任意其他盘中建立项目文件夹，在项目文件夹中创建`do_intel.bat`文件，其内容如下（注意这里使用的是ifx，24年之后intel推荐使用此编译器代替ifort）：
```
set OPTC=-I"C:\Program Files (x86)\Microsoft SDKs\MPI\Include\intel"
set OPTL="C:\Program Files (x86)\Microsoft SDKs\MPI\Lib\x64\msmpi.lib" "C:\Program Files (x86)\Microsoft SDKs\MPI\Lib\x64\msmpifec.lib"
call ifx %OPTC% -o %1.exe %1.f90 %OPTL%
```
在项目文件夹中再创建`runmpi.bat`文件，其内容如下：
```
set OPT="C:\Program Files\Microsoft MPI\bin"
%OPT%\mpiexec -n %2 %1
```
如果需要测试，可在项目文件夹中再创建一个`test.f90`文件：
```
program main
   use mpi
   implicit none

   integer size, rank, tag
   integer stat(mpi_status_size), ierr
   integer I
   real R

   call mpi_init( ierr )
   call mpi_comm_size( mpi_comm_world, size, ierr )
   call mpi_comm_rank( mpi_comm_world, rank, ierr )
   write( *, * ) "Processor ", rank, " of ", size, " is alive"

   tag = 0
   if ( rank == 0 ) then
      I = 42
      R = 3.14152
      call mpi_send( I, 1, mpi_integer, 1, tag, mpi_comm_world, ierr )
      write( *, * ) "Processor ", rank, " sent integer ", I
      call mpi_send( R, 1, mpi_real   , 1, tag, mpi_comm_world, ierr )
      write( *, * ) "Processor ", rank, " sent real ", R

   else if ( rank == 1 ) then
      call mpi_recv( I, 1, mpi_integer, 0, tag, mpi_comm_world, stat, ierr )
      write( *, * ) "Processor ", rank, " received integer ", I
      call mpi_recv( R, 1, mpi_real, 0, tag, mpi_comm_world, stat, ierr )
      write( *, * ) "Processor ", rank, " received real ", R
   end if

   call mpi_finalize( ierr )

end program main
```
## 7. 测试是否成功。
以管理员身份打开`Intel oneAPI command prompt for IA32 for Visual Studio 2022`，cd到项目文件夹。使用`do_intel test`命令编译test.f90。再使用`runmpi test.exe 2`运行并行程序，这里使用了2个进程。
## 8. 在vscode中安装C/C++, Code Runner, Modern Fortran插件。
## 9. 在vscode中创建可编译ifort和ifx的终端。
在vscode窗口下方选择终端，点加号默认会打开一个Powershell终端，这里需要将默认终端替换为之前使用过的oneAPI command prompt。为实现这一目的，点击加号旁边的箭头，在下拉菜单中选择“选择默认配置文件”。此时会在窗口上面列出已有的终端配置，让选择一个。鉴于我们要新配置一个终端，而其是基于CMD打开的，所以点击CMD右面的齿轮标志。命名新终端为OneAPI然后回车。使用文件-->打开文件来打开vscode的`settings.json`文件，其默认位置一般为`C:\Users\<username>\AppData\Roaming\Code\User\settings.json`。找到`OneAPI`代码块，在其中的变量行args后的中括号中加入：
```
"cmd /E:ON /K",
"C:\\Program Files (x86)\\Intel\\oneAPI\\setvars.bat"
```
这表明在打开CMD后立即执行这条命令，事实上，如果在之前的第5步和第7步中不使用`Intel oneAPI command prompt for IA32 for Visual Studio 2022`而是直接使用CMD，就需要先运行`cmd /E:ON /K "C:\\Program Files (x86)\\Intel\\oneAPI\\setvars.bat"`这一串命令后，才能用ifort和ifx进行编译。在`terminal.integrated.profiles.windows`代码块后面加上`"terminal.integrated.defaultProfile.windows": "OneAPI"`，将OneAPI设为默认终端。注意每行结尾要打逗号。
## 10. 修改Modern Fortran的编译器。
Modern Fortran的编译器默认为gfortran，其对MPI的支持度不太好，需要修改为ifort或者ifx。在刚才设置默认终端的代码后加上以下两行：
```
"fortran.linter.compiler":"ifx" // 设置Fortran的默认编译器为ifx
"fortran.linter.compilerPath": "C:\\Program Files (x86)\\Intel\\oneAPI\\compiler\\latest\\bin\\ifx" // 指定编译器所在路径
```
这样即可在vscode的终端中直接进行MPI代码的编译了。

## 参考网址：
- [VS2019+MSMPI+Intel OneAPI编译运行Fortran并行程序](https://blog.csdn.net/PilotJohnWu/article/details/120636428).
- [Fortran-部署基于VS Code的Fortran开发环境](https://www.cnblogs.com/ziangshen/articles/16633516.html).
- [【高性能计算】完美解决Windows下安装mpi环境并应用到VSCode中报错问题的方法](https://blog.csdn.net/weixin_45942927/article/details/125167460).
- [Using MS Visual Studio Code, how do I access an MPI library in a Fortran code?](https://stackoverflow.com/questions/75725750/using-ms-visual-studio-code-how-do-i-access-an-mpi-library-in-a-fortran-code).
- [Fortran MPI using Cygwin64](https://stackoverflow.com/questions/75444805/fortran-mpi-using-cygwin64/75449313#75449313).
- [How to Set Up Intel Fortran Development on Windows in Visual Studio & Visual Studio Code](https://gist.github.com/hballington12/615f43c403ed1f5d5de2bb2e9b8f44ed).
- [How to use ifort (+ fpm) on Windows?](https://fortran-lang.discourse.group/t/how-to-use-ifort-fpm-on-windows/4540).
