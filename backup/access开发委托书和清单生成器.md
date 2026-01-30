### 开发背景
一闲下来就爱折腾这折腾那的 ，想着把Python 代码开发的 业务小软件转成access 使用 没想到 效果比python 开发的还要好点
文件大小又小 只需要 数据库就行 ，唯一不足的就是清单生成器 盖章的问题 没有什么好的解决办法，不过不影响使用  暂时先这样吧 
看习惯了就好  ，为什么开发这套工具？ 由于同行的业务票都找我开 有的要委托书和清单表 ，有时候 填写不方便 尤其是大小写的问题 经常不注意就会出错 于是先萌生了使用 python开发一套业务系统  
### 下面把代码部分贴出来备份 
委托书部分VBA代码
```
Option Explicit

' ============================================================
' 配置区
' ============================================================
Const STAMP_PATH As String = "E:\办公文档\印章\公章.png" ' <-- 请确保路径下有透明底色的 PNG 公章
Const FIXED_UNIT As String = "名称"

Sub Deploy_Final_System()
    Dim frm As Form, mdl As Module, strName As String
    Dim cbo As ComboBox, btn As CommandButton
    
    ' 1. 初始化数据库表
    On Error Resume Next
    DoCmd.DeleteObject acTable, "tblAccounts"
    On Error GoTo 0
    CurrentDb.Execute "CREATE TABLE tblAccounts (AccName TEXT(50), Bank TEXT(100), AccNum TEXT(100))"
    
    ' 插入预设账户
    CurrentDb.Execute "INSERT INTO tblAccounts VALUES ('姓名', '银行', '账号')"
    CurrentDb.Execute "INSERT INTO tblAccounts VALUES  ('姓名', '银行', '账号')"
    CurrentDb.Execute "INSERT INTO tblAccounts VALUES ('姓名', '银行', '账号')"

    ' 2. 创建 UI 窗体
    Set frm = CreateForm: strName = frm.Name
    frm.Caption = "委托书生成系统 (自动盖章版)"
    
    CreateField strName, "txtObject", "委托对象:", 400
    CreateField strName, "txtService", "业务内容:", 900
    CreateField strName, "txtAmountLow", "金额(小写):", 1400
    CreateField strName, "txtAmountUp", "金额(大写):", 1900
    
    ' 账户下拉框
    Set cbo = CreateControl(strName, acComboBox, acDetail, , , 1600, 2400, 4500, 350)
    cbo.Name = "cboAcc": cbo.RowSource = "SELECT AccName, Bank, AccNum FROM tblAccounts"
    cbo.ColumnCount = 3: cbo.ColumnWidths = "1440;2500;0"
    
    ' 生成按钮
    Set btn = CreateControl(strName, acCommandButton, acDetail, , , 1600, 3100, 3000, 600)
    btn.Name = "btnGen": btn.Caption = "★ 生成 Word 并自动盖章"
    btn.OnClick = "[Event Procedure]"

    ' 3. 注入逻辑
    Set mdl = frm.Module
    ' 自动转换大写
    mdl.AddFromString "Private Sub txtAmountLow_AfterUpdate()" & vbCrLf & _
                      "    Me.txtAmountUp = CNMoney(Val(Nz(Me.txtAmountLow, 0)))" & vbCrLf & _
                      "End Sub"
    
    ' 注入 Word 核心引擎
    mdl.AddFromString GetStrictWordEngine()
    ' 注入大写转换算法
    mdl.AddFromString GetAlgoCode()

    DoCmd.Restore: DoCmd.OpenForm strName
    MsgBox "部署成功！" & vbCrLf & "请确保 " & STAMP_PATH & " 文件存在。", vbInformation
End Sub

' --- Word 核心生成引擎 ---
Function GetStrictWordEngine() As String
    Dim s As String
    s = "Private Sub btnGen_Click()" & vbCrLf
    s = s & "  Dim wApp As Object, wDoc As Object, r As Object, shp As Object" & vbCrLf
    s = s & "  On Error Resume Next" & vbCrLf
    s = s & "  Set wApp = CreateObject(""Word.Application""): wApp.Visible = True" & vbCrLf
    s = s & "  Set wDoc = wApp.Documents.Add" & vbCrLf
    
    s = s & "  ' --- 红色红线标题 ---" & vbCrLf
    s = s & "  Set r = wDoc.Range(0, 0)" & vbCrLf
    s = s & "  With r" & vbCrLf
    s = s & "    .Text = ""委托书"" & vbCrLf: .Font.Name = ""方正小标宋简体"": .Font.Size = 26" & vbCrLf
    s = s & "    .Font.Color = 255: .ParagraphFormat.Alignment = 1" & vbCrLf
    s = s & "    .ParagraphFormat.Borders(-3).LineStyle = 1: .ParagraphFormat.Borders(-3).LineWidth = 12: .ParagraphFormat.Borders(-3).Color = 255" & vbCrLf
    s = s & "  End With" & vbCrLf
    
    s = s & "  ' --- 正文内容 ---" & vbCrLf
    s = s & "  Set r = wDoc.Range(wDoc.Range.End - 1)" & vbCrLf
    s = s & "  With r" & vbCrLf
    s = s & "    .InsertAfter vbCrLf & ""兹委托："" & Me.txtObject & vbCrLf & vbCrLf" & vbCrLf
    s = s & "    .InsertAfter ""    将"" & Me.txtService & ""款项人民币 "" & Me.txtAmountLow & "" 元，""" & vbCrLf
    s = s & "    .InsertAfter ""人民币大写金额：（"" & Me.txtAmountUp & ""）打入以下账户："" & vbCrLf & vbCrLf" & vbCrLf
    s = s & "    .InsertAfter ""    户  名："" & Me.cboAcc.Column(0) & vbCrLf" & vbCrLf
    s = s & "    .InsertAfter ""    开户行："" & Me.cboAcc.Column(1) & vbCrLf" & vbCrLf
    s = s & "    .InsertAfter ""    账  号："" & Me.cboAcc.Column(2) & vbCrLf" & vbCrLf
    s = s & "    .Font.Name = ""仿宋_GB2312"": .Font.Size = 16: .Font.Color = 0" & vbCrLf
    s = s & "    .ParagraphFormat.Alignment = 0: .ParagraphFormat.LineSpacing = 28" & vbCrLf
    s = s & "  End With" & vbCrLf
    
    s = s & "  ' --- 落款日期 ---" & vbCrLf
    s = s & "  Set r = wDoc.Range(wDoc.Range.End - 1)" & vbCrLf
    s = s & "  With r" & vbCrLf
    s = s & "    .InsertAfter vbCrLf & vbCrLf & ""委托单位："" & """ & FIXED_UNIT & """ & vbCrLf" & vbCrLf
    s = s & "    .InsertAfter Format(Date, ""yyyy 年 mm 月 dd 日"")" & vbCrLf
    s = s & "    .ParagraphFormat.Alignment = 2" & vbCrLf
    s = s & "  End With" & vbCrLf
    
    s = s & "  ' --- 核心：模拟正片叠底盖章 ---" & vbCrLf
    s = s & "  If Dir(""" & STAMP_PATH & """) <> """" Then" & vbCrLf
    s = s & "    Set shp = wDoc.Shapes.AddPicture(""" & STAMP_PATH & """, False, True)" & vbCrLf
    s = s & "    With shp" & vbCrLf
    s = s & "      .WrapFormat.Type = 3 ' wdWrapBehind (衬于文字下方，文字会透出来)" & vbCrLf
    s = s & "      .ZOrder 5            ' 确保在文字底层" & vbCrLf
    s = s & "      .Width = 110: .Height = 110" & vbCrLf
    s = s & "      .RelativeHorizontalPosition = 1 ' 相对于页面" & vbCrLf
    s = s & "      .RelativeVerticalPosition = 1" & vbCrLf
    s = s & "      .Left = 360  ' 调整此值控制左右位置" & vbCrLf
    s = s & "      .Top = 520   ' 调整此值控制上下位置，使其盖住日期" & vbCrLf
    s = s & "    End With" & vbCrLf
    s = s & "  End If" & vbCrLf
    s = s & "End Sub"
    GetStrictWordEngine = s
End Function

' --- 辅助：人民币大写转换 ---
Function GetAlgoCode() As String
    Dim a As String
    a = "Function CNMoney(ByVal n As Double) As String" & vbCrLf
    a = a & "  Dim d, u, i, v, p, z, r: d = Array(""零"",""壹"",""贰"",""叁"",""肆"",""伍"",""陆"",""柒"",""捌"",""玖"")" & vbCrLf
    a = a & "  u = Array("""",""拾"",""佰"",""仟"",""万"",""拾"",""佰"",""仟"",""亿"")" & vbCrLf
    a = a & "  p = CStr(Fix(n)): z = False" & vbCrLf
    a = a & "  For i = 1 To Len(p): v = Val(Mid(p, i, 1)): r = Len(p) - i" & vbCrLf
    a = a & "    If v <> 0 Then" & vbCrLf
    a = a & "      If z Then CNMoney = CNMoney & ""零"": z = False" & vbCrLf
    a = a & "      CNMoney = CNMoney & d(v) & u(r)" & vbCrLf
    a = a & "    Else: z = True: If r Mod 4 = 0 And r > 0 Then CNMoney = CNMoney & u(r)" & vbCrLf
    a = a & "    End If" & vbCrLf
    a = a & "  Next: CNMoney = IIf(CNMoney="""",""零"",CNMoney) & ""圆整""" & vbCrLf
    a = a & "End Function"
    GetAlgoCode = a
End Function

' --- 辅助：创建 UI 控件 ---
Private Sub CreateField(f As String, n As String, lTxt As String, y As Long)
    Dim t As textbox, l As Label
    Set l = CreateControl(f, acLabel, acDetail, , , 100, y, 1400, 350): l.Caption = lTxt
    Set t = CreateControl(f, acTextBox, acDetail, , , 1600, y, 3500, 350): t.Name = n
End Sub
```         
清单部分VBA代码
```
Option Explicit

' ============================================================
' 配置区：请确保路径正确
' ============================================================
Const STAMP_PATH As String = "E:\办公文档\印章\公章.png"
Const COMPANY_NAME As String = "名称"

' ============================================================
' 1. 部署系统 (点击此过程并按 F5 运行)
' ============================================================
Sub Deploy_Professional_System()
    Dim frm As Form, mdl As Module, strName As String
    
    ' 清理并重新创建数据表
    On Error Resume Next: DoCmd.DeleteObject acTable, "tblServiceDetails": On Error GoTo 0
    CurrentDb.Execute "CREATE TABLE tblServiceDetails (ID COUNTER PRIMARY KEY, SDate DATE, SItem TEXT(255), SUnit TEXT(50), SQty DOUBLE, SPrice CURRENCY)"

    ' 创建窗体
    Set frm = CreateForm: strName = frm.Name
    frm.Caption = "名称"
    frm.width = 8500
    
    ' 创建输入字段
    CreateField strName, "txtDate", "服务日期:", 400, "Date()"
    CreateField strName, "txtItem", "项目内容:", 900
    CreateField strName, "txtUnit", "计量单位:", 1400, "'次'"
    CreateField strName, "txtQty", "服务数量:", 1900, "1"
    CreateField strName, "txtPrice", "单价(元):", 2400
    
    ' 创建功能按钮
    With CreateControl(strName, acCommandButton, acDetail, , , 1600, 3100, 1400, 500)
        .Name = "btnAdd": .Caption = "添加项目": .OnClick = "[Event Procedure]"
    End With
    With CreateControl(strName, acCommandButton, acDetail, , , 3100, 3100, 1400, 500)
        .Name = "btnClear": .Caption = "清空清单": .OnClick = "[Event Procedure]"
    End With
    With CreateControl(strName, acCommandButton, acDetail, , , 1600, 3800, 2900, 700)
        .Name = "btnExport": .Caption = "确定 (生成盖章PDF)": .OnClick = "[Event Procedure]"
    End With

    ' 注入逻辑代码
    Set mdl = frm.Module: mdl.AddFromString GetFinalEngine()
    
    DoCmd.Restore: DoCmd.OpenForm strName
End Sub

' ============================================================
' 2. 核心代码引擎 (仅修复印章定位至总计金额右侧，其余逻辑不变)
' ============================================================
Function GetFinalEngine() As String
    Dim s As String
    
    s = "Private Sub btnAdd_Click()" & vbCrLf & _
        "  If IsNull(Me.txtItem) Or IsNull(Me.txtPrice) Then MsgBox ""请填写内容"": Exit Sub" & vbCrLf & _
        "  CurrentDb.Execute ""INSERT INTO tblServiceDetails (SDate, SItem, SUnit, SQty, SPrice) VALUES (#"" & Me.txtDate & ""#, '"" & Me.txtItem & ""', '"" & Me.txtUnit & ""', "" & Me.txtQty & "", "" & Me.txtPrice & "")""" & vbCrLf & _
        "  Me.txtItem = """": Me.txtQty = 1: Me.txtItem.SetFocus" & vbCrLf & _
        "End Sub" & vbCrLf

    s = s & "Private Sub btnClear_Click()" & vbCrLf & _
        "  If MsgBox(""确认清空？"", vbQuestion + vbYesNo) = vbYes Then CurrentDb.Execute ""DELETE FROM tblServiceDetails"": MsgBox ""已清空""" & vbCrLf & _
        "End Sub" & vbCrLf

    s = s & "Private Sub btnExport_Click()" & vbCrLf
    s = s & "  Dim wApp As Object, wDoc As Object, wTable As Object, rs As Object" & vbCrLf
    s = s & "  Dim totalAll As Double, filePath As String, i As Integer" & vbCrLf
    s = s & "  Set rs = CurrentDb.OpenRecordset(""SELECT * FROM tblServiceDetails"")" & vbCrLf
    s = s & "  If rs.EOF Then MsgBox ""没有数据"": Exit Sub" & vbCrLf
    s = s & "  filePath = CreateObject(""WScript.Shell"").SpecialFolders(""Desktop"") & ""\结算单_"" & Format(Now, ""mmdd_HHMM"") & "".pdf""" & vbCrLf
    
    s = s & "  Set wApp = CreateObject(""Word.Application""): wApp.Visible = False" & vbCrLf
    s = s & "  Set wDoc = wApp.Documents.Add" & vbCrLf
    s = s & "  With wDoc.PageSetup: .LeftMargin = 60: .RightMargin = 60: .TopMargin = 50: End With" & vbCrLf

    ' 标题和表格绘制
    s = s & "  wDoc.Range(0, 0).Text = """ & COMPANY_NAME & """ & vbCrLf & ""服务结算清单"" & vbCrLf" & vbCrLf
    s = s & "  With wDoc.Paragraphs(1).Range: .Font.Size = 18: .Font.Bold = True: .ParagraphFormat.Alignment = 1: End With" & vbCrLf
    s = s & "  rs.MoveLast: i = rs.RecordCount: rs.MoveFirst" & vbCrLf
    s = s & "  Set wTable = wDoc.Tables.Add(wDoc.Paragraphs(wDoc.Paragraphs.Count).Range, i + 1, 5)" & vbCrLf
    s = s & "  wTable.Borders.Enable = False: wTable.Rows(1).Borders(-3).LineStyle = 1" & vbCrLf
    s = s & "  wTable.Cell(1, 1).Range.Text = ""服务日期"": wTable.Cell(1, 2).Range.Text = ""项目名称""" & vbCrLf
    s = s & "  wTable.Cell(1, 3).Range.Text = ""数量"": wTable.Cell(1, 4).Range.Text = ""单价"": wTable.Cell(1, 5).Range.Text = ""小计""" & vbCrLf
    
    s = s & "  i = 2: totalAll = 0" & vbCrLf
    s = s & "  Do While Not rs.EOF" & vbCrLf
    s = s & "    wTable.Cell(i, 1).Range.Text = Format(rs!SDate, ""yyyy/mm/dd"")" & vbCrLf
    s = s & "    wTable.Cell(i, 2).Range.Text = rs!SItem" & vbCrLf
    s = s & "    wTable.Cell(i, 3).Range.Text = rs!SQty & "" "" & rs!SUnit" & vbCrLf
    s = s & "    wTable.Cell(i, 4).Range.Text = Format(rs!SPrice, ""0.00"")" & vbCrLf
    s = s & "    wTable.Cell(i, 5).Range.Text = Format(rs!SQty * rs!SPrice, ""0.00"")" & vbCrLf
    s = s & "    totalAll = totalAll + (rs!SQty * rs!SPrice): i = i + 1: rs.MoveNext" & vbCrLf
    s = s & "  Loop" & vbCrLf
    
    ' 生成合计行，并获取该行的对象作为印章锚点
    s = s & "  Dim targetPara As Object" & vbCrLf
    s = s & "  wDoc.Paragraphs.Add" & vbCrLf
    s = s & "  Set targetPara = wDoc.Paragraphs(wDoc.Paragraphs.Count)" & vbCrLf
    s = s & "  With targetPara.Range" & vbCrLf
    s = s & "    .Text = vbCrLf & ""-----------------------------------------"" & vbCrLf & ""合计总金额：￥ "" & Format(totalAll, ""#,##0.00"") & "" """ & vbCrLf
    s = s & "    .Font.Bold = True: .Font.Size = 14: .ParagraphFormat.Alignment = 2" & vbCrLf
    s = s & "  End With" & vbCrLf
    
    ' --- 终极印章修复：精准绑定总计金额右侧，永久锁定位置 ---
    s = s & "  If Dir(""" & STAMP_PATH & """) <> """" Then" & vbCrLf
    s = s & "    Dim stamp As Object, totalTextRange As Object" & vbCrLf
    s = s & "    Set totalTextRange = targetPara.Range" & vbCrLf
    ' 锚点绑定到合计金额文字最后一个字符，杜绝偏移
    s = s & "    Set stamp = wDoc.Shapes.AddPicture(""" & STAMP_PATH & """, False, True, totalTextRange.Characters(totalTextRange.Characters.Count))" & vbCrLf
    s = s & "    With stamp" & vbCrLf
    s = s & "      .WrapFormat.Type = 3" & vbCrLf ' 浮于文字上方（保留原设置）
    s = s & "      .LockAspectRatio = True: .Width = 110" & vbCrLf ' 保留原尺寸
    s = s & "      .RelativeHorizontalPosition = 2" & vbCrLf ' 相对于字符（精准跟随金额文字）
    s = s & "      .RelativeVerticalPosition = 2" & vbCrLf ' 相对于字符（垂直对齐金额文字）
    s = s & "      .Left = 10" & vbCrLf ' 紧贴总计金额右侧（微小偏移，无间隙）
    s = s & "      .Top = -(.Height / 2) + 2" & vbCrLf ' 垂直居中于金额文字行（微调贴合）
    s = s & "      .LockAnchor = True" & vbCrLf ' 永久锁定锚点，防止印章移位
    s = s & "    End With" & vbCrLf
    s = s & "  End If" & vbCrLf

    s = s & "  wDoc.ExportAsFixedFormat filePath, 17" & vbCrLf
    s = s & "  wDoc.Close 0: wApp.Quit" & vbCrLf
    s = s & "  MsgBox ""PDF 生成完毕！印章已定位。"", vbInformation" & vbCrLf
    s = s & "End Sub"
    
    GetFinalEngine = s
End Function

' --- 辅助函数：创建窗体控件 ---
Private Sub CreateField(f As String, n As String, lTxt As String, y As Long, Optional defVal As String = "")
    Dim t As textbox, l As Label
    Set l = CreateControl(f, acLabel, acDetail, , , 400, y, 1100, 350): l.Caption = lTxt
    Set t = CreateControl(f, acTextBox, acDetail, , , 1600, y, 3500, 350): t.Name = n
    If defVal <> "" Then t.DefaultValue = defVal
End Sub


···
