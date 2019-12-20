4.1 修改md命令

完成此任务只需要增加一个判断当前目录下是否存在同名文件的分支即可。在`MdComd()`函数中判断新目录名是否符合规则后，调用`FindFCB()`判断是否存在同名目录或文件，若存在则报错退出，若不存在则继续处理。

4.2 修改命令行预处理程序

完成此任务只需要增加处理"cd.."的逻辑，这里不考虑输出重定向功能。在`ParseCommand()`函数中分解命令及其参数后，对`comd[0]`逐字符判断是否存在".."，若存在则将其分解为两个参数，之前的参数依次后移一个位置即可。

4.3 新增fc命令，实现两个文件的比较

将两个文件的内容分别保存到`buffer`中，对`buffer`进行比较。在`FcComd()`函数中，首先判断命令参数数量是否为2，若不是则报错退出。接下来调用`ProcessPath()`和`FindFCB()`函数，判断文件的存在性以及构造文件的全路径名，若两个文件的全路径名相同，则报错不能自比较。最后比较文件长度，调用`file_to_buffer()`函数将文件内容保存到`buffer`中，对`buffer`逐字符比较内容。

4.4 新增replace命令，实现文件取代

将源文件内容保存到`buffer`中，利用`buffer_to_file()`函数修改目标文件内容。在`ReplaceComd()`函数中，首先判断参数提供情况然后做出相应提示处理。接下来调用`ProcessPath()`和`FindFCB()`函数，判断文件的存在性以及构造文件的全路径名。若被取代文件为只读属性文件，询问用户是否继续操作。若被取代文件具有隐藏和系统属性，则报错提示。若两文件为同一文件，则报错提示。判断文件属性可以使用位运算：

```C++
if ((int)fcbp2->Fattrib & 1)
	{
		cout << "\n文件" << gFileName2 << "是只读属性的文件。是否继续操作？（y/n）\n";
		char choice;
		cin >> choice;
		if (choice == 'N' || choice == 'n')
			return;
	}
if ((int)fcbp2->Fattrib & 6)
	{
		cout << "\n文件" << gFileName2 << "具有隐藏和系统属性的文件不能被取代。\n";
		return;
	}
```

最后调用`file_to_buffer()`将源文件内容保存到`buffer`中，再调用`buffer_to_file()`覆盖目标文件内容。

4.6 新增batch命令，实现批处理

此任务分为两种处理模式：

​	①从真实磁盘中读入文件处理

​	②从模拟磁盘中读入文件处理

在`BatchComd()`函数中根据参数数量来区别两种处理模式，若参数数量为1则为情况①，若参数数量为2则为情况②，其他情况为参数错误。

对于情况①，使用`ifstream`输入文件流即可完成。即将`main()`函数中的标准输入流改成文件输入流：

```c++
ifstream fin(comd[1], ios::in);
if (!fin.is_open())
{
    cout << "\n文件" << comd[1] << "打开失败。\n";
    return;
}
while (!fin.eof())	//循环，直到文件结束
{			//输入命令，分析并执行命令
    cout << "\nC:";					//显示提示符(本系统总假定是C盘)
    if (dspath)
        cout << curpath.cpath;
    cout << ">";
    fin.getline(cmd, INPUT_LEN);		//输入命令
    cout << cmd << endl;
    if (strlen(cmd))
    {
        k = ParseCommand(cmd);		//分解命令及其参数
        //comd[0]中是命令，comd[1],comd[2]...是参数		
        ExecComd(k);				//执行命令
    }
}
fin.close();
```

对于情况②，首先判断`comd[2]`是否为's',若不是则报错退出。然后采取的处理方式是调用`ProcessPath()`,`FindFCB()`判断文件存在性，构造文件全路径名以及获得文件目录项指针，再调用`file_to_buffer()`保存到`buffer`中，对`buffer`进行分析。当处理到'\n'时，完成一条命令的输入即可。

4.7 修改close、type等命令，允许不带文件名参数

为了完成此任务，增加了一个`CurFile`的数据结构：

```c++
struct CurFile		//定义当前操作文件的数据结构
{
	char cpath[PATH_LEN];		//当前操作文件绝对路径字符串(全路径名)
};
```

增加了一个修改当前操作文件的函数：

```c++
void ChangeCurFile(char* path)
{
	strcpy(curFile.cpath, path);
}
```

当涉及命令不提供相应参数时，提示将对当前操作文件进行操作。例如在`CloseComd()`函数中：

```c++
if (k < 1)
{
    cout << "\n命令中缺少文件名。将使用当前操作文件。\n";
    strcpy(comd[1], curFile.cpath);
}
```

同时在涉及命令成功操作后更新当前操作文件。例如在`WriteComd()`函数中：

```c++
cout << "\n写文件" << uof[ii_uof].fname << "成功.\n";
ChangeCurFile(temppath);
return 1;
```

4.8 修改del、copy等命令，使其可以使用统配符 *（选做内容）

采用递归的设计思想处理此任务。当发现涉及命令参数出现统配符'*'时，首先定位当前目录，由于同目录文件`FCB`的连续性，可以使用循环来处理当前目录下的所有文件。当遇到非空目录项时，构造相应的命令参数，然后调用相应的命令即可完成此文件的处理。

例如在`AttribComd()`函数中发现`comd[1]`为"\*"后，进入统配符处理逻辑。首先构造出一个泛文件路径（例如'/usr/\*'），然后调用`ProcessPath()`定位当前目录获得目录首块号`s`，根据实际情况这里需要对根目录特殊处理。接下来对`s`循环处理，即在首块号为`s`的目录寻找非空登记栏，直到目录尾部。当找到非空目录项时，构造相应命令参数后再调用目标函数：

```c++
while (s > 0)			//在首块号为s的目录找非空登记栏，直到目录尾部
{
    FCB* fcbp1 = (FCB*)Disk[s];
    for (i = 0; i < 16; i++, fcbp1++)
        if (fcbp1->FileName[0] != (char)0xe5 && fcbp1->FileName[0] != '\0')
        {
            char* FilePath = new char[PATH_LEN];
            strcpy(FilePath, cur_path);
            int len = strlen(FilePath);
            if (FilePath[len - 1] != '/')
                strcat(FilePath, "/");
            strcat(FilePath, fcbp1->FileName);
            strcpy(comd[1], FilePath);
            //cout << comd[1] << endl;
            AttribComd(k);//找到非空目录项
            delete[] FilePath;
        }
    s0 = s;		//记下上一个盘块号
    s = FAT[s];	//取下一个盘块号
}
```

4.14 修改磁盘块容量（选做内容）

磁盘块大小由64字节修改为256字节，主要修改之处在于对每个盘块读取目录项的循环次数由4改为16，除此以外并无大的改动。例如：

```c++
for (i = 0; i < 16; i++, fcbp1++)
```

4.15 修改undel命令（选做内容）

采用3种修改方案中的第2种，配合4.14 修改磁盘块容量一起完成。

首先修改`UnDel`数据结构：

```c++
struct UnDel		//恢复被删除文件表的数据结构(共128字节,110+11+1+2+2+2=128)
{
	char gpath[110];		//被删除文件的全路径名(不含文件名)
	char ufname[FILENAME_LEN];	//被删除文件名
	char ufattr;				//被删除文件属性
	short ufaddr;				//被删除文件名的首块号
	short fb;					//存储被删除文件块号的第一个块号(链表头指针)
								//首块号也存于fb所指的盘块中
	short ufsize;				//被删除文件大小
};
```

然后更改`udtab`的声明：

```c++
UnDel* udtab;				//定义删除文件恢复表
```

在`main()`函数中完成`udtab`的定义：

```c++
int b0 = 4980;
udtab = (UnDel*)Disk[b0];
short* pp = (short*)Disk[0];
ffbp = pp[0];
Udelp = 0;
```

原始命令只能恢复目录项没有被占用的被删除文件，同时要求被删除文件磁盘块也未被占用。增强功能修改后，在`UnDel`结构中保存被删除文件的所有信息，便于处理目录项被占用而磁盘块未被占用的恢复情况。

同时修改`undel`命令的处理逻辑：对`udtab`表中的所有表项逐个进行处理，恢复时首先在对应目录中寻找被删除目录项进行恢复。若未找到被删除目录项则确定发生了该文件目录项被占用的情况，此时新建一个空白目录项进行恢复即可。

例如在`UndelComd()`函数中，使用`for`循环逐个处理表项。首先用`FindPath()`判断表项路径是否正确并获得目录首块号`s`。增加`bool flag=false`,当循环遍历目录后未找到被删除文件目录项时，进入额外恢复处理逻辑：

```c++
if (!flag)//未搜索到匹配目录项 即目录项已经被占用 新建目录项恢复
{
    cout << "\n文件"<<udtab[i].ufname<<"原目录项已经被占用，是否仍要恢复？（y/n）\n";
    char choice;
    cin >> choice;
    if (choice == 'y' || choice == 'Y')
    {
        FindBlankFCB(s0, fcbp1);
        fcbp1->Addr = udtab[i].ufaddr, fcbp1->Fattrib = udtab[i].ufattr, fcbp1->Fsize = udtab[i].ufsize;
        strcpy(fcbp1->FileName, udtab[i].ufname);
        fcbp1->FileName[0] = (char)0xe5;
        fn = &(fcbp1->FileName[1]);
        Udfile(fcbp1, s0, fn, cc);
    }
}
```

通过调用`FindBlankFCB()`来新建目录项，然后调用`Udfile()`来完成文件恢复。

若在循环遍历目录中找到被删除文件目录项，则置`flag`为`true`,然后调用`Udfile()`来完成文件恢复。

