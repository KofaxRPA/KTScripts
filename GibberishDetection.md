# Gibberish Detection Script  
Pages that are processed in Kofax Transformation may not contain intelligible language because of scanning problems, image quality or obfuscation occuring in a pdf document.
This script detects whether the document contains "readable language".   
1. Create a Dictionary  
1. Create locators to test a document with the dictionary
## Create a Dictionary
 * Load a set of documents into Project Builder that contain the kind of language you need. Add as many documents as possible. hundreds or thousands are better.
 * Switch the Document Viewer to **Hierarchy View**  
 ![image](https://user-images.githubusercontent.com/47416964/77313755-bc059280-6d04-11ea-84f1-5b17bc604894.png)
 * Swtich the **Runtime Script Events** to Event **Batch_Close**  
 ![image](https://user-images.githubusercontent.com/47416964/77313830-ddff1500-6d04-11ea-837e-e075b11f5876.png)
 ![image](https://user-images.githubusercontent.com/47416964/77313869-ec4d3100-6d04-11ea-97b3-f62afb6b3deb.png)
 * Copy the script below to the Project Class
 * In the Script Debugger **Edit/References...** a refrence to **Microsoft Scripting Runtime** so you can use the **Dictionary** object
 * Execute the script by clicking the **Lightning** icon.
![image](https://user-images.githubusercontent.com/47416964/77313939-0850d280-6d05-11ea-85d0-b1351fbc1731.png)
*This script can process 100 documents in about 10 seconds. It removes punctation, ignores numbers and stores words that are 2 or more characters long*




 
```VBA
Const digits="0123456789"
Const punc="!£$%^&*()<>,.;:'@[]#{}\|/"","
Private Sub Batch_Close(ByVal pXRootFolder As CASCADELib.CscXFolder, ByVal CloseMode As CASCADELib.CscBatchCloseMode)
   Dim X As Long, XDoc As CscXDocument, Dict As New Dictionary, W As Long, Word As String, key As String, Count As Long, DocCount As Long, c As Long, ch As String
   Dim numeric As Boolean

   For X=0 To pXRootFolder.DocInfos.Count-1
      Set XDoc=pXRootFolder.DocInfos(X).XDocument
      For W=0 To XDoc.Words.Count-1
         Word=LCase(XDoc.Words(W).Text)
         numeric=False
         For c=1 To Len(digits)
            numeric=InStr(Word,Mid(digits,c,1))
            If numeric Then Exit For
         Next
         If Not numeric Then
            For c=1 To Len(punc)
               Word=Replace(Word,Mid(punc,c,1),"")
            Next
            If Len(Word)>1 Then
               If Dict.Exists(Word) Then
                  Dict(Word)=Dict(Word)+1
               Else
                  Dict.Add(Word,1)
               End If
            End If
        End If
      Next
   Next
   DocCount=pXRootFolder.DocInfos.Count
   Open "c:\temp\english.dict" For Output As 1
   Print #1, vbUTF8BOM;
   For Each key In Dict.Keys
      Count=Dict(key)
      If Count>DocCount/10 Then 'if the word appears in at least 10% of documents
         Print #1, key & "," & Format(Len(key),"#") & "-" & Format (Count/DocCount,"0.000")
      End If
   Next
   Close #1
End Sub

Private Sub SL_English_LocateAlternatives(ByVal pXDoc As CASCADELib.CscXDocument, ByVal pLocator As CASCADELib.CscXDocField)
   Dim Words As CscXDocFieldAlternatives, W As Long
   Set Words=pXDoc.Locators.ItemByName("FL_English").Alternatives
   With pLocator.Alternatives.Create
      .Text="English"
      For W=0 To Words.Count-1
         .Confidence=.Confidence+ CInt(Split(Words(W).Text,"-")(0))*CDbl(Split(Words(W).Text,"-")(1))
      Next
      For W=0 To pXDoc.Words.Count-1
         .LongTag=.LongTag+Len(pXDoc.Words(W).Text)
      Next
      .Confidence=.Confidence/.LongTag
   End With
End Sub
```
