
# 采用GO实现的FTP协议
FTP协议的GO版本实现，目前实现了大部分常规FTP命令的处理，包括FTP服务端和客户端，通信框架底层采用[https://github.com/suiyunonghen/DxTcpServer](https://github.com/suiyunonghen/DxTcpServer "DxTcpServer框架")。
# 简要说明
> go get github.com/suiyunonghen/DxCommonLib
> go get github.com/suiyunonghen/DxTcpServer
> go get github.com/suiyunonghen/DxNetProtocols


# FtpServer用法
```go
package main
import (
	"github.com/suiyunonghen/DxNetProtocols/Ftp"
	"github.com/suiyunonghen/GVCL/Components/Controls"
	"github.com/suiyunonghen/GVCL/Components/NVisbleControls"
	"github.com/suiyunonghen/GVCL/WinApi"
	"unsafe"
	"syscall"
)

func main()  {
	app := controls.NewApplication()
	srv := Ftp.NewFtpServer()

	//设置anonymouse可以下载
	srv.SetAnonymouseFilePermission(true,true,true,false)

	//FTP登录的时候，新增用户,赋予密码以及权限等
	srv.OnGetFtpUser = func(userId string) *Ftp.FtpUser {
		if userId == "DxSoft"{
			result := new(Ftp.FtpUser)
			result.UserID = userId
			result.PassWord = "DxSoft"
			//赋值权限为匿名用户权限
			srv.CopyAnonymousUserPermissions(result)
			result.Permission.SetFileWritePermission(true)
			return result
		}
		return nil
	}

	app.ShowMainForm = false
	mainForm := app.CreateForm()
	PopMenu := NVisbleControls.NewPopupMenu(mainForm)
	mItem := PopMenu.Items().AddItem("服务信息")
	mItem.OnClick = func(sender interface{}) {
		//通过网页返回服务端消息
		WinApi.ShellExecute(mainForm.GetWindowHandle(),uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr("OPEN"))),
			uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr("https://github.com/suiyunonghen"))),0,0,WinApi.SW_SHOWNORMAL)
	}

	mItem = PopMenu.Items().AddItem("-")
	mItem = PopMenu.Items().AddItem("退出")
	mItem.OnClick = func(sender interface{}) {
		srv.Close()
		mainForm.Close()
	}
	trayicon := NVisbleControls.NewTrayIcon(mainForm)
	trayicon.PopupMenu = PopMenu
	trayicon.SetVisible(true)
	//在GUI运行之前，开启文件服务功能
	srv.MapDir("FtpDir1","F:\\Web",true)
	//srv.MapDir("FtpDir2","H:\\FtpDir2",false)
	srv.Open(":8340")
	app.Run()
}
```
# 说明
FTPServer包含方法
>MapDir(remotedir,localPath string,isMainRoot bool)
>用来添加映射FTP的目录路径，remotedir指定远程路径名，映射到本地的localPath作为FTP的一个目录，isMainRoot指定是否为FTP的主目录


>SetAnonymouseFilePermission(canRead,canWrite,canAppend,canDelete bool)
>设定anonymouse账户的文件访问权限


>SetAnonymouseDirPermission(canCreateDir,canDelDir,canListDir,canSubDirs bool)
>设定anonymouse账户的目录访问权限

>CopyAnonymousUserPermissions(user *FtpUser)
>拷贝anonymouse账户权限到FtpUser中，一般是初始化FTPUser的时候会用到

>WelcomeMessage用来指定FTP服务的欢迎消息
>PublicIP指定Pasv模式时候，外放IP
>MinPasvPort和MaxPasvPort指定Pasv模式的端口范围
>OnGetFtpUser事件在用户登录FTP账户的时候，如果内部账户列表中未发现的，则会触发本事件，用来判定是否有效的账户给定


# FtpClient用法
package main

import (
	"github.com/suiyunonghen/DxNetProtocols/Ftp"
	"fmt"
	"time"
	"github.com/suiyunonghen/GVCL/Components/Controls"
	"github.com/suiyunonghen/GVCL/Components/NVisbleControls"
	"github.com/suiyunonghen/GVCL/WinApi"
	"unsafe"
	"syscall"
)

func main()  {
	app := controls.NewApplication()
	client := Ftp.NewFtpClient(Ftp.TDM_PASV)
	client.OnResultResponse = func(responseCode uint16, msg string) {
		fmt.Println("ResponseInfo:  ",responseCode,"  ",msg)
	}

	client.OnDataProgress = func(TotalSize int, CurPos int, TransTimes time.Duration, transOk bool) {
		if TransTimes == -1{
			fmt.Println("开始传输")
		}else if transOk{
			fmt.Println("传输完成，共传输数据：",TotalSize," 传输时长：",TransTimes)
		}
	}

	app.ShowMainForm = false
	mainForm := app.CreateForm()
	PopMenu := NVisbleControls.NewPopupMenu(mainForm)
	mItem := PopMenu.Items().AddItem("服务信息")
	mItem.OnClick = func(sender interface{}) {
		//通过网页返回服务端消息
		WinApi.ShellExecute(mainForm.GetWindowHandle(),uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr("OPEN"))),
			uintptr(unsafe.Pointer(syscall.StringToUTF16Ptr("https://github.com/suiyunonghen"))),0,0,WinApi.SW_SHOWNORMAL)
	}

	mItem = PopMenu.Items().AddItem("连接FTP")
	mItem.OnClick = func(sender interface{}) {
		if !client.Active(){
			client.Connect("127.0.0.1:8340")



			client.ListDir("html", func(ftpFileinfo *Ftp.FTPFile) {
				fmt.Println(ftpFileinfo)
			})

			if err := client.Login("DxSoft","DxSoft");err == nil{
				fmt.Println("login OK")
			}else{
				fmt.Println("login failed:",err)
			}
			if _,err := client.ExecuteFtpCmd("PWD","",nil);err != nil{
				return
			}
			client.ExecuteFtpCmd("OPTS","UTF8 ON",nil)
		}
		//获取目录信息
		client.ExecuteFtpCmd("CWD","html",nil)
		client.ExecuteFtpCmd("CWD","html",nil)

		fmt.Println("List: ")

		client.ListDir("html", func(ftpFileinfo *Ftp.FTPFile) {
			fmt.Println(ftpFileinfo)
		})

		client.ListDir("/html5Test/html5test/css", func(ftpFileinfo *Ftp.FTPFile) {
			fmt.Println(ftpFileinfo)
		})

		client.ListDir("/html5Test/html5test/css", func(ftpFileinfo *Ftp.FTPFile) {
			fmt.Println(ftpFileinfo)
		})

		client.DownLoad("HoorayOS 2.0.0.zip","d:\\tt.zip",0)
		client.UpLoad("D:\\tools\\10.23更新-单文件版 自带SVIP.exe","自带SVIP.exe",0)
	}

	mItem = PopMenu.Items().AddItem("-")
	mItem = PopMenu.Items().AddItem("退出")
	mItem.OnClick = func(sender interface{}) {
		mainForm.Close()
	}
	trayicon := NVisbleControls.NewTrayIcon(mainForm)
	trayicon.PopupMenu = PopMenu
	trayicon.SetVisible(true)
	app.Run()
}

# 说明
主要提供客户端的访问处理，其实主要函数就一个ExecuteFtpCmd

>ExecuteFtpCmd(cmd,param string,checkResultOk CheckCmdOkFunc)(responspkg *ftpResponsePkg, e error)
>执行FTP命令，cmd表示命令，Param表示参数，其中checkResultOk是一个函数，主要用来判定这条命令是否完整返回，如果一条命令只有一条返回结果的话，可以设定为nil，否则需要自己设定验证，比如LIST等需要传输数据的指令，一般可能会有2条指令返回，所以这里就需要用来做处理，来设定本命令是否完整结束。
>Connect(addr string)error连接到FTPServer
>Login(uid,pwd string)error登录FTP服务
>ListDir(dirName string,listfunc func(ftpFileinfo *FTPFile))
>获取目录的文件列表
>DownLoad(remoteFileName,localFileName string,fromPos int)error
>从FTP上下载remoteFileName到本地的localFileName，从fromPos位置开始
>UpLoad(localFile,remoteFile string,fromPos int)error 
>将本地的localFile上传到FTP的本目录下的remoteFile，从fromPos开始
>其他的指令请全部参照使用ExecuteFtpCmd来进行完成。
