# vs2017与vs2005冲突解决

先安装vs2005，后安装vs2017，vs2017无法运行
原因：vs2005注册了Microsoft.VisualStudio.Shell.Interop.8.0.dll，优先于vs2017需要的dll。导致运行错误。

解决方法：

复制
C:\Program_Files_(x86)\Microsoft_Visual Studio\2017\Enterprise\Common7\IDE\PublicAssemblies\Microsoft.VisualStudio.Shell.Interop.8.0.dll
到
C:\Windows\assembly\GAC\Microsoft.VisualStudio.Shell.Interop.8.0\8.0.0.0__b03f5f7f11d50a3a\Microsoft.VisualStudio.Shell.Interop.8.0.dll
