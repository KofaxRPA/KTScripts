```vb
'#Language "WWB-COM"

Option Explicit
'This script uses all unique words on a page to register OCR and OMR zones to subpixel accuracy.
'!!! IMPORTANT !!!!
' Add on Menu/Edit/References... "Microsoft Scripting Runtime" for Dictionary
'Create Locators
'  SL_UniqueWords (used for debugging so you can see the unique words)
'  SL_CalculatePageShift (with subfields M, B, Smoothness, N, Resolution, Direction
'  SL_MoveZones (this moves all the OCR and OMR zones for the AZL)
'  AZL (on the Registration Tab set to "None")

Private Sub SL_UniqueWords_LocateAlternatives(ByVal pXDoc As CASCADELib.CscXDocument, ByVal pLocator As CASCADELib.CscXDocField)
   'Your document MUST be classified before calling this locator, in order to be able to find the sample image in the AZL.
   'This function is purely here for debugging. it is so that you can see the unique words that are used for matching
   Dim I As Long, StartWordIndexRef As Long, StartWordIndex As Long, EndWordIndexRef As Long, EndWordIndex As Long
   Dim AZLSampleDoc As CscXDocument, LeftShift As Double, DownShift As Double, Tolerance As Double, Confidence As Double
   Dim AZLSampleDocFileName As String
   AZLSampleDocFileName =Left(Project.FileName,InStrRev(Project.FileName,"\")) & "Samples\" & Class_GetClassPath(pXDoc.ExtractionClass) & "\Sample0.xdc"
   Set AZLSampleDoc = New CscXDocument
   AZLSampleDoc.Load(AZLSampleDocFileName)
   For I=0 To pXDoc.Pages.Count - 1
      Pages_Compare(AZLSampleDoc.Pages(I),pXDoc.Pages(I),pLocator.Alternatives,pXDoc.CDoc.Pages(I).XRes,pXDoc.CDoc.Pages(I).YRes)
   Next
End Sub

Private Sub SL_CalculatePageShift_LocateAlternatives(ByVal pXDoc As CASCADELib.CscXDocument, ByVal pLocator As CASCADELib.CscXDocField)
   'this works out the page shift between the pages
   Dim P As Long
   Dim AZLSampleDoc As CscXDocument, LeftShift As Double, DownShift As Double, Tolerance As Double, Confidence As Double
   Dim AZLSampleDocFileName As String
   AZLSampleDocFileName =Left(Project.FileName,InStrRev(Project.FileName,"\")) & "Samples\" & Class_GetClassPath(pXDoc.ExtractionClass) & "\Sample0.xdc"
   Set AZLSampleDoc = New CscXDocument
   AZLSampleDoc.Load(AZLSampleDocFileName)
   For P=0 To pXDoc.Pages.Count - 1
      Pages_Compare(AZLSampleDoc.Pages(P),pXDoc.Pages(P),pLocator.Alternatives,pXDoc.CDoc.Pages(P).XRes,pXDoc.CDoc.Pages(P).YRes)
   Next
End Sub

Private Function Class_GetClassPath(ClassName As String) As String
   'Recursively work out the ClassPath
   Dim ParentClass As CscClass
   Set ParentClass=Project.ClassByName(ClassName).ParentClass
   If ParentClass Is Nothing Then
      Return ClassName
   Else
      Return Class_GetClassPath(ParentClass.Name) & "\" & ClassName
   End If
End Function

Private Sub SL_MoveZones_LocateAlternatives(ByVal pXDoc As CASCADELib.CscXDocument, ByVal pLocator As CASCADELib.CscXDocField)
   'Move the Zones in the Advanced Zone Locator based on the Shifts,scale and smoothness
   'Add reference to "Kofax Advanced Zone Locator 4.0"
   Dim Shifts As CscXDocFieldAlternatives
   Dim Zones As CscXDocSubFields, AZL As CscAdvZoneLocator
   Set Shifts=pXDoc.Locators.ItemByName("SL_CalculatePageShift").Alternatives
   Set AZL = Project.ClassByName(pXDoc.ExtractionClass).Locators.ItemByName("AZL").LocatorMethod
   Zones_Shift(AZL.Zones,Shifts,pXDoc.Representations(0))
End Sub

Public Sub Pages_Compare(page1 As CscXDocPage, page2 As CscXDocPage,Results As CscXDocFieldAlternatives, XRes As Long, YRes As Long)
   'Find as many unique anchors between the two pages and work out the shift and scaling between them
   'This algorithm is very robust against OCR errors.
   'The algorithm has order of page.words.count, i.e. linear, so VERY FAST.
   Dim Words1 As Dictionary, Words2 As Dictionary, WordText As String, X() As Long, Y() As Long
   Dim Word1 As CscXDocWord, Word2 As CscXDocWord
   Dim VectorField As CscXDocField, Result As CscXDocFieldAlternative, B As Long, C As Long
   Dim Vectors As CscXDocFieldAlternatives, Vector As CscXDocFieldAlternative
   Set VectorField=New CscXDocField
   Set Vectors=VectorField.Alternatives
   Set Words1=Page_GetUniqueWords(page1,0,page1.Words.Count-1)
   Set Words2=Page_GetUniqueWords(page2,0,page2.Words.Count-1)
   'Build a list of the unique words that appear on BOTH pages
   For Each WordText In Words1.Keys
     If Len(WordText) >= 6  And IsNumeric(WordText) = False Then 'only match words with 6 or more characters
         If Words2.Exists(WordText) Then 'This unique word appears on both pages
            Set Word1=Words1(WordText)
            Set Word2=Words2(WordText)
            Set Vector=Vectors.Create
            Vector.Left=Word1.Left+Word1.Width/2
            Vector.Top=Word1.Top+Word1.Height/2
            Vector.Width=Word2.Left+Word2.Width/2
            Vector.Height=Word2.Top+Word2.Height/2
            Vector.Text=Word1.Text
         End If
    End If
   Next
   LinearRegression(Vectors,True,Results.Create,XRes,Results.Count-1) 'Calculate horizontal shift, scale and smoothness
   LinearRegression(Vectors,False,Results.Create,YRes,Results.Count-1) 'Calculate vertical shift, scale and smoothness
End Sub

Public Function Page_GetUniqueWords(page As CscXDocPage,StartWordIndex As Long,EndWordIndex As Long) As Dictionary
   'Add Reference to "Microsoft Scripting Runtime" for Dictionary
   'Find all words on the page that only appear once
   Dim w As Long, Word As CscXDocWord, WordText As String
   Dim Words As New Dictionary
   For w=StartWordIndex To EndWordIndex
      Set Word=page.Words(w)
      If Words.Exists(Word.Text) Then
         Set Words(Word.Text) = Nothing 'this word is not unique
      Else
         Words.Add(Word.Text,Word)
      End If
   Next
   For Each WordText In Words.Keys 'Remove the non-unique words
      If Words(WordText) Is Nothing Then Words.Remove(WordText)
   Next
   Return Words
End Function

Public Sub LinearRegression(Vectors As CscXDocFieldAlternatives, Vector As Boolean, Result As CscXDocFieldAlternative, Resolution As Long,AlternativeIndex As Double)
   'http://en.wikipedia.org/wiki/Simple_linear_regression' https://www.easycalculation.com/statistics/learn-regression.php
   'The 1st Alternative has the horizonatal Scaling=M, displacement=B, and Confidence=flatness of the paper.
   'The 2nd Alternative has the vertical    Scaling=M, displacement=B, and Confidence=flatness of the paper.
   Dim X As Double, Y As Double, Sx As Double, Sy As Double, Sxy As Double, Sxx As Double, Syy As Double, V As Long
   Dim B As Double, M As Double, N As Long, R As Double
   For V= 0 To Vectors.Count-1
      If Vector Then
         X=Vectors(V).Left
         Y=Vectors(V).Width
      Else
         X=Vectors(V).Top
         Y=Vectors(V).Height
      End If
      Sx=Sx+X
      Sy=Sy+Y
      Sxy=Sxy+X*Y
      Sxx=Sxx+X^2
      Syy=Syy+Y^2
   Next
   N=Vectors.Count
   M=(N*Sxy-Sx*Sy)/(N*Sxx-Sx^2)  'slope of linear regression
   B=(Sy-M*Sx)/N                 'y intercept of linear regression
   R=(N*Sxy-Sx*Sy)/Sqr((N*Sxx-Sx^2)*(N*Syy-Sy^2))  'correlation 1.00=perfect fit= smooth paper
   With Result.SubFields.Create("M")
      .Confidence=M
      .Text=Format(M,"0.000")
   End With
   With Result.SubFields.Create("B")
      .Confidence=B
      .Text=Format(B,"0.000")
   End With
   With Result.SubFields.Create("Smoothness")
      .Confidence=R
      .Text=Format(R,"0.0000")
   End With
   With Result.SubFields.Create("N")
      .Confidence=N
      .Text=CStr(N)
   End With
   With Result.SubFields.Create("Resolution")
      .Confidence=Resolution
      .Text=CStr(Resolution)
   End With
   With Result.SubFields.Create("Direction")
      .Confidence=1
      .Text=IIf(Vector,"Horizontal","Vertical")
   End With

   ' Result.Confidence=IIf(Vector,1.1,1.0)

'this was done To maintain the Right order For All the pages. Each page will have 2 alternatives (Horizontal And Vertical)
   Result.Confidence=1.0-(AlternativeIndex*0.000001)
End Sub

Public Sub Zones_Shift(AZLZones As CscAdvZoneLocZones, Shifts As CscXDocFieldAlternatives, Rep As CscXDocRepresentation)
   'Add reference to
   Dim Z As Long, XDocZone As CscXDocZone
   While Rep.Zones.Count>0
      Rep.Zones.Remove(0)
   Wend
   For Z=0 To AZLZones.Count-1
      Set XDocZone=Zone_Shift(AZLZones(Z),Shifts,Rep)
      Rep.Zones.Append(XDocZone)
   Next
End Sub

Public Function Zone_Shift(AZLZone As CscAdvZoneLocZone, Shifts As CscXDocFieldAlternatives, Rep As CscXDocRepresentation) As CscXDocZone
   Dim XDocZone As CscXDocZone, X As Double, Y As Double, Right As Long, Bottom As Long
   Set XDocZone=New CscXDocZone
   XDocZone.PageNr=AZLZone.PageNr
   XDocZone.Name=AZLZone.Name
   'Shift the top right corner
   X=AZLZone.Left+AZLZone.Width
   Y=AZLZone.Top
   Coordinate_Shift(X,Y,Shifts,AZLZone.PageNr)
   Right=X
   'Shift the bottom left corner
   X=AZLZone.Left
   Y=AZLZone.Top+AZLZone.Height
   Coordinate_Shift(X,Y,Shifts,AZLZone.PageNr)
   Bottom=Y
   'Shift the Top Left corner
   X=AZLZone.Left
   Y=AZLZone.Top
   Coordinate_Shift(X,Y,Shifts,AZLZone.PageNr)
   XDocZone.Left=X
   XDocZone.Top=Y
   XDocZone.Width=Right-XDocZone.Left
   XDocZone.Height=Bottom-XDocZone.Top
   Return XDocZone
End Function

Public Sub Coordinate_Shift(ByRef X As Double, ByRef Y As Double, Shifts As CscXDocFieldAlternatives, page As Integer)
   Dim XRes As Long, YRes As Long, xm As Double, xb As Double, ym As Double, yb As Double
   With Shifts(page*2)
      xm=.SubFields.ItemByName("M").Confidence
      xb=.SubFields.ItemByName("B").Confidence
      XRes=.SubFields.ItemByName("Resolution").Confidence
   End With
   With Shifts(page*2+1)
      ym=.SubFields.ItemByName("M").Confidence
      yb=.SubFields.ItemByName("B").Confidence
      YRes=.SubFields.ItemByName("Resolution").Confidence
   End With
   X=X/25.4*XRes
   Y=Y/25.4*YRes
   X=xm*X+xb  'The Linear regression function gave us these slopes m and intercepts b.
   Y=ym*Y+yb
End Sub
```
