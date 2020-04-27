**Table of Contents**

<!-- toc -->

- [How to Read Russian Invoices in Kofax Transformation](#how-to-read-russian-invoices-in-kofax-transformation)
  * [How to read Russian Tables](#how-to-read-russian-tables)
    + [Detecting Table Headers](#detecting-table-headers)
    + [Correcting Table Values](#correcting-table-values)
  * [Amount formatter for - and =](#amount-formatter-for---and-)
  * [Locating INN & KPP](#locating-inn--kpp)
  * [Split INN and KPP](#split-inn-and-kpp)
  * [INN Checksum Algorithm](#inn-checksum-algorithm)
  * [Quick-Correct of Numerical fields.](#quick-correct-of-numerical-fields)
  * [Check that the net, tax and total under the table actually match the sum of the table columns](#check-that-the-net-tax-and-total-under-the-table-actually-match-the-sum-of-the-table-columns)
  * [Useful functions](#useful-functions)
  * [Format Invoice Number](#format-invoice-number)
  * [Spell Check Country Names](#spell-check-country-names)
  * [Units Formatting](#units-formatting)

<!-- tocstop -->

# How to Read Russian Invoices in Kofax Transformation
Russian invoice have some unique components, that are different from a typical European or American invoice.
* [INN](https://www.nalog.ru/eng/exchinf/inn/) (10 or 12 digit Taxpayer Personal Identification Number, with checksum)  
* KPP (9 digit Tax Registration Event Code) Numbers for both vendor and customer.
* very wide tables  for line items that have between 11 and 15 columns. These columns are very regular and well defined. The last row of the table header contains the column number.  
![image](https://user-images.githubusercontent.com/47416964/80201852-0f7d4000-8625-11ea-96f6-e1343728dead.png)  
* The table total and tax information is embedded inside the last row of the table
* Russian invoices can use **-** as a decimal separator and **=** as a negative sign. eg "=101-00" = "-100.00"

## How to read Russian Tables
Russian Table headers have lots of words with considerable word wrapping. This is a challenge to the table locator.
The following script detects accuractely the Table header. It uses fuzzy logic to avoid OCR errors.
1. Detect the textline containing ""1 2 3 4 5 6 7 8 9 10 11 12" using fuzzy logic
1. Assign the columns based on the words above the columns, using fuzzy logic.
1. Detect the end of table with a dictionary fuzzily looking for *всего к оплате* (Total Payable) and it's variants.
```
Итого
Всего к оплате
Итого по НДС
Итого по листу
Итого по ставке 
Всего
ВСЕГО ПО 
```
4. Cluster textlines within the table into table rows to deal with line wrapping.
1. Insert all words in the table into the correct cells
1. Repair OCR errors in numbers using the mathematical relationships
  * **Quantity** * **Unit Price** = **Net Price** (q*u=n)
  * **Net** * **TaxRate** = **TaxAmount** (n*r=x)
  * **Net** + **TaxAmount** = **Total** (n+x=t)
  if one amount is has an OCR error then it can be reconstructed using the above three rules  
  
| q | u | n | r | x | t |              |
| - | - | - | - | - | - | -----------  |
| * | * | * |   |   |   | q*u=n        |
|   |   | * |   | * | * | n+x=t        |
| * | * |   |   | * | * | q*u+x=t      |
| * | * |   | * |   |   | q\*u*(1+r)=t  |
|   |   | * | * |   | * | n*(1+r)=t    |
|   |   |   | * | * | * | x(1+1/r)=t   |

### Detecting Table Headers
Add the following to a dictionary file with substitutions and add it to a format locator, and in place of a regex add the dictionary reference. This locator will identify all the words in the headers and label them.
```
headerword;columns
^дрда^сапр;Net_Amount
^единица;Unit_Measure
^имущественных;Net_Amount
^налога;Total_Price
«нш1ы«е;Unit_Measure
•л;Total_Price
1мущсствс1пых;Net_Amount
1рав;Description
1том;Total_Price
1х;Total_Price
1ца;Unit_Measure
1циф;Country_Of_Origin_Code
1чество;Quantity
1я;Unit_Measure
6м;Total_Price
i;Unit_Price,Net_Amount,Description,Customs_Declaration,Unit_Measure
№;Description,Position
а;Description
аединицу;Unit_Price
азанных;Description
ак-;Excise
акц;Excise
акци;Excise
акциз;Excise
акциза;Excise
акциэ;Excise
акш13;Excise
акшрз;Excise
аможей-•;Customs_Declaration
ана;Country_Of_Origin,Country_Of_Origin_Code
арти-;Article_Code
артикула;Article_Code
аьтполиенних;Description
ая;Tax_Amount,Tax_Rate
б;Excise
без;Net_Amount
бот;Total_Price
в;Excise
в5;Unit_Measure
валю;Currency
валюты;Currency
вание;Currency
вара;Article_Code
вая;Tax_Rate
венных;Net_Amount,Total_Price
веного;Description
во;Quantity
вой;Country_Of_Origin_Code
всего;Net_Amount,Total_Price
всогосучетои;Total_Price
вщика;Article_Code
вы;Description
вылолненных;Description
вып;Description
выполненных;Description
выполнены;Description
выполненых;Description
вьшолненшх;Description
г;Description,Country_Of_Origin
-г";Quantity
говая;Tax_Rate
говея;Tax_Rate
гроиосож-;Country_Of_Origin
гроисхож;Country_Of_Origin
д;Country_Of_Origin
дек;Customs_Declaration
декла;Customs_Declaration
деклар;Customs_Declaration
деклара-ц;Customs_Declaration
декларации;Customs_Declaration
декларацик;Customs_Declaration
декпарач;Customs_Declaration
дения;Country_Of_Origin
дех;Customs_Declaration
е;Excise
е^е^иницу;Unit_Price
ед;Unit_Measure,Unit_Price
еди;Unit_Measure
еди-;Unit_Measure
един;Unit_Price,Unit_Measure
едини;Unit_Measure
единиц;Unit_Price
единица;Unit_Measure,Unit_Of_Measure_Code
единицу;Unit_Price
единицу-;Unit_Price
ел;Unit_Price
енных;Net_Amount
ждения;Country_Of_Origin
женной;Customs_Declaration
за;Unit_Price
зава;Description
заед;Unit_Price
зая;Tax_Rate
и2мераньц1;Unit_Price
йалогом;Total_Price
иаме-;Unit_Measure
иг;Description
ие;Country_Of_Origin
из;Unit_Measure
из-;Unit_Measure
изм;Unit_Price,Unit_Measure
изме;Unit_Measure
изме-;Unit_Price,Unit_Measure
измер;Unit_Price
измере;Unit_Price
измере-;Unit_Price
измёре-;Unit_Measure
измерен;Unit_Price
изме-рен;Unit_Measure
измерения;Unit_Price,Unit_Of_Measure_Code,Unit_Measure
измс-;Unit_Measure
ии;Customs_Declaration,Country_Of_Origin
ииущипвенньи;Net_Amount
йия;Unit_Measure
ику-;Total_Price
именование;Description
иму-;Net_Amount
иму1цествен;Total_Price
имуирственных;Total_Price
имуцеотвен-;Total_Price
имущ;Net_Amount,Total_Price
имущвстаеквого;Description
имущесгцо1л10п;Description
имущест;Net_Amount,Total_Price
имущест-;Description,Net_Amount,Total_Price
имуществ;Total_Price,Net_Amount
имуществе;Description
имуществен;Net_Amount,Total_Price
имущественн;Net_Amount,Total_Price
имущественно;Description
имущественного;Description
имущественною;Description
имущественные;Description
имущественных;Total_Price,Net_Amount,Description
имуще-ственных;Net_Amount
имуществправ;Net_Amount
имуществрогр;Description
имущостбонньп;Total_Price
имущправ;Net_Amount,Total_Price
иниэм;Unit_Price
иного;Description
инуществен-;Total_Price
исх;Country_Of_Origin
исхождения;Country_Of_Origin
ица;Unit_Measure
иэ;Unit_Measure
иэм;Unit_Measure
ия;Unit_Measure,Unit_Price
к<к;Unit_Of_Measure_Code
ка;Tax_Rate
казанных;Description
каименовавие;Description
канмбнова1ие;Description
кие;Currency
кипа;Unit_Measure
код;Unit_Of_Measure_Code,Country_Of_Origin_Code,Article_Code
кож-;Country_Of_Origin
кол-;Quantity
кол-во;Quantity
коли;Quantity
коли-;Quantity
количе;Quantity
количество;Quantity
коп;Quantity
коп-;Quantity
копи-;Quantity
краткое;Country_Of_Origin
кула1покупа-;Article_Code
л;Description
лё;Excise
лекпараши;Customs_Declaration
лого^;Tax_Rate
локупа;Tax_Amount
лрав;Total_Price
лрава;Description
лрвдьяв;Tax_Amount
лссго;Total_Price
луг;Net_Amount,Total_Price,Description
ля;Article_Code
ляеман;Tax_Amount
ляемая;Tax_Amount
малого;Tax_Rate
ме;Unit_Measure
меженной;Customs_Declaration
мма;Tax_Amount
на-;Description
над;Unit_Measure
надо-;Tax_Rate
наиманоыание;Description
наиме;Currency
найме;Currency
найме-;Currency
наимен;Currency
наименаваяиет&вара;Description
наимено;Currency
наиме-но;Currency
наименова;Country_Of_Origin
наименован;Country_Of_Origin
наименование;Description,Country_Of_Origin
наименований;Description
наименоеание;Description
нал;Net_Amount,Tax_Rate,Total_Price
нало;Tax_Rate
нало-;Tax_Rate
налог;Tax_Rate
налога;Net_Amount,Tax_Amount,Total_Price
налога";Net_Amount
налоге;Total_Price
налого;Tax_Rate
налого-;Tax_Rate
нало-го;Tax_Rate
налогов;Tax_Rate
налоговая;Tax_Rate
налогом;Total_Price
нальное;Unit_Measure
нацио;Unit_Measure
национально;Unit_Measure
национальное;Unit_Measure
не-;Unit_Measure
нелогом-;Total_Price
ни;Customs_Declaration
ние;Country_Of_Origin,Currency
нил;Country_Of_Origin
нио;Country_Of_Origin
них;Net_Amount
ница;Unit_Measure
ния;Unit_Measure,Country_Of_Origin,Unit_Price
ннца;Unit_Measure
нова;Currency
нование;Currency
нойкетарации;Customs_Declaration
ном;Customs_Declaration
номер;Customs_Declaration,Article_Code
номфр;Article_Code
нрава;Description
нчвс1ъеного;Description
ных;Net_Amount,Total_Price
ньгх;Total_Price
нэп;Tax_Rate
о;Total_Price,Description,Tax_Rate
оаание;Currency
обоза;Unit_Measure
обозна;Unit_Measure
обозначение;Unit_Measure
объем;Quantity
объём;Quantity
ов;Total_Price
огаэанныкуслуг;Description
ого;Tax_Rate
огоикость;Net_Amount
огоимость;Total_Price
ождени;Country_Of_Origin
ой;Customs_Declaration
оказанных;Description
оказе^нных;Description
окезашшх;Description
олнсюгис;Description
опи-;Description
описание;Description
описаний;Description
описаяие;Description
орав;Total_Price
от;Country_Of_Origin
п;Description,Position
пало-;Tax_Rate
пмушёствеилых;Total_Price
по4агелю;Tax_Amount
покупа;Tax_Amount
покупате;Article_Code
покупатели;Tax_Amount
покупателю;Tax_Amount
покупателя;Article_Code
поливных;Description
полненньа;Description
полога;Tax_Amount
пра;Total_Price
праа;Net_Amount,Total_Price
прав;Net_Amount,Total_Price,Description
права;Description
праввсего;Total_Price
праэ;Total_Price
предъяв;Tax_Amount
предъявляв;Tax_Amount
предъявляем;Tax_Amount
предъявляемая;Tax_Amount
про;Country_Of_Origin
про1;Country_Of_Origin
проиохож;Country_Of_Origin
проис;Country_Of_Origin,Country_Of_Origin_Code
проис-;Country_Of_Origin
происх;Country_Of_Origin
происхо;Country_Of_Origin
происхож;Country_Of_Origin
происхож-;Country_Of_Origin
происхожден;Country_Of_Origin_Code,Country_Of_Origin
происхождения;Country_Of_Origin_Code,Country_Of_Origin
происхозденяя;Country_Of_Origin_Code,Country_Of_Origin
пронсхож-;Country_Of_Origin
псста-;Article_Code
р-;Total_Price
р^бот;Net_Amount
ра;Total_Price
ра5ог;Description
ра6от;Net_Amount
ра6отуспуг;Total_Price
раб;Total_Price
работ;Description,Net_Amount,Total_Price
работ^ока;Description
работоказанных;Description
работус-;Total_Price
работуслуг;Net_Amount,Total_Price
работуспуг;Total_Price
рабст;Net_Amount
рабуо;Net_Amount
рации;Customs_Declaration
ре-;Unit_Measure
реи;Unit_Measure
рен;Unit_Measure
рени;Unit_Measure
рения;Unit_Measure,Unit_Price
реп;Unit_Measure
риф;Unit_Price
ров;Net_Amount,Total_Price
роисхож-;Country_Of_Origin
ртрана;Country_Of_Origin
с;Total_Price,Description
с^мма;Tax_Amount
сание;Description
спуг;Total_Price
ст;Tax_Rate
ста4;Tax_Rate
став;Tax_Rate
став-;Tax_Rate
ставка;Tax_Rate
ство;Quantity
—стижшстб—;Total_Price
стйиыйсть;Net_Amount
стойкость;Total_Price
стоимосп;Total_Price
стоймост;Net_Amount
стоимость;Net_Amount,Total_Price
стоимтова;Net_Amount,Total_Price
стоимтоваров;Net_Amount,Total_Price
стокмтоваров-;Total_Price
стр;Country_Of_Origin,Country_Of_Origin_Code
страна;Country_Of_Origin,Country_Of_Origin_Code
страна-;Country_Of_Origin
страну;Country_Of_Origin
ст-ть;Net_Amount,Total_Price
стцигапстб;Net_Amount
сумма;Tax_Amount,Excise
сучетнал;Total_Price
схож-;Country_Of_Origin
та;Customs_Declaration,Unit_Price
та-;Unit_Price
таиожойной;Customs_Declaration
там;Customs_Declaration
тамо;Customs_Declaration
тамож;Customs_Declaration
таможенно;Customs_Declaration
таможенной;Customs_Declaration
таможсшюг;Customs_Declaration
таножен-н;Customs_Declaration
тараф;Unit_Price
тариф;Unit_Price
тел;Tax_Amount
телю;Tax_Amount
теля;Article_Code
тнагп;Tax_Rate
тоаара;Description
тоааров;Total_Price
тов;Net_Amount
това;Description,Total_Price
това-;Article_Code
товар;Total_Price
товара;Description,Country_Of_Origin_Code,Country_Of_Origin
товаров;Net_Amount,Total_Price
товврог;Total_Price
тои;Excise
той;Excise
том;Excise,Total_Price
тон;Excise
трф^ёг;Description
туг;Description
тч;Excise
тч^;Excise
ты;Currency
тэможевн;Customs_Declaration
у;Description,Unit_Price
уепуг;Total_Price
унетом;Total_Price
ус;Unit_Measure,Net_Amount
ус-i;Net_Amount
усл;Unit_Measure,Net_Amount,Total_Price
условное;Unit_Measure
услуг;Description,Net_Amount,Total_Price
успимущ;Net_Amount
успуг;Description
успуп;Total_Price
успцг;Net_Amount
уч;Total_Price
уче;Total_Price
уче-;Total_Price
учет;Total_Price
учетом;Total_Price
учётом;Total_Price
хожде-;Country_Of_Origin
хождения;Country_Of_Origin_Code,Country_Of_Origin
ца;Unit_Measure
цбка;Unit_Price
це1и;Unit_Price
цена;Unit_Price
ценз;Unit_Price
цера;Unit_Price
цеяа;Unit_Price
циз;Excise
цифр;Country_Of_Origin_Code
цйфро?;Country_Of_Origin_Code
цифровой;Country_Of_Origin_Code
цче-;Total_Price
ч;Total_Price
ч^ймтогтаой-;Net_Amount
чение;Unit_Measure
чеотао;Quantity
чест;Quantity
чество;Quantity
честео;Quantity
чис;Excise
чйс;Excise
числ;Excise
числд;Excise
числе;Excise
чисре;Excise
чсегво;Quantity
шалото-;Tax_Rate
щественых;Net_Amount
щестеенных;Total_Price
щуг;Description
ых;Description
ь;Net_Amount
ь1х;Net_Amount
эа;Unit_Price
элкеных;Description
эписание;Description
юна;Country_Of_Origin
я;Country_Of_Origin_Code,Country_Of_Origin,Unit_Measure
```
### Correcting Table Values
This corrects all numerical values according to formuale above, along with spellchecking and correcting country names.
```vbscript
Private Sub CorrectCells(ByVal pXDoc As CscXDocument, ByVal Table As CscXDocTable)
   Dim r As Integer
   Dim c As Integer
   Dim cf As ICscFieldFormatter
   Dim uf As ICscFieldFormatter

   Set cf = Project.FieldFormatters.ItemByName("CountryNameFormatter")
   Set uf = Project.FieldFormatters.Item("UnitsFormatter")
   For r = 0 To Table.Rows.Count - 1
      TableRow_CorrectAmounts (Table.Rows(r), tolerance)
      With Table.Rows(r)
         uf.FormatTableCell (.Cells.ItemByName("Unit Measure"))
         cf.FormatTableCell (.Cells.ItemByName("Country Of Origin"))
         'Set all empty cells and error-free cells to valid
         For c = 0 To Table.Columns.Count - 1
            If Table.Rows(r).Cells(c).Text = "" Or Table.Rows(r).Cells(c).ErrorDescription = "" Then Table.Rows(r).Cells(c).ExtractionConfident = True
         Next
      End With
   Next
End Sub

Public Sub TableRow_CorrectAmounts(row As CscXDocTableRow,tol As Double)
   Dim afl As ICscFieldFormatter 'Amount Formatter
   Dim pfl As ICscFieldFormatter 'Percent Formatter
   Set afl=Project.FieldFormatters.ItemByName(Project.DefaultAmountFormatter)
   Set pfl=Project.FieldFormatters.ItemByName("PercentageFormatter")
   Dim q,u,n,r,x,t As CscXDocTableCell
   Set q=row.Cells.ItemByName("Quantity")
   Set u=row.Cells.ItemByName("Unit Price")
   Set n=row.Cells.ItemByName("Net Amount")
   Set r=row.Cells.ItemByName("Tax Rate")
   Set x=row.Cells.ItemByName("Tax Amount")
   Set t=row.Cells.ItemByName("Total Price")
   afl.FormatTableCell(q)
   afl.FormatTableCell(u)
   afl.FormatTableCell(n)
   pfl.FormatTableCell(r)
   afl.FormatTableCell(x)
   afl.FormatTableCell(t)
   Dim qun,nxt,nrt,rxt,nxr,quxt,validTaxRate As Boolean
   validTaxRate=(r.DoubleValue=10 Or r.DoubleValue=18)
   If q.DoubleValue>0 And u.DoubleValue>0 And n.DoubleValue>0                     AndAlso Abs(q.DoubleValue*u.DoubleValue              -n.DoubleValue)<tol Then qun =True
   If n.DoubleValue>0 And x.DoubleValue>0 And t.DoubleValue>0                     AndAlso Abs(n.DoubleValue+x.DoubleValue              -t.DoubleValue)<tol Then nxt =True
   If n.DoubleValue>0 And validTaxRate    And t.DoubleValue>0                     AndAlso Abs(n.DoubleValue*(1+r.DoubleValue/100)      -t.DoubleValue)<tol Then nrt =True
   If validTaxRate    And x.DoubleValue>0 And t.DoubleValue>0                     AndAlso Abs(x.DoubleValue*(1+100/r.DoubleValue)      -t.DoubleValue)<tol Then rxt =True
   If n.DoubleValue>0 And x.DoubleValue>0 And validTaxRate                        AndAlso Abs(n.DoubleValue*r.DoubleValue/100          -x.DoubleValue)<tol Then nxr =True
   If q.DoubleValue>0 And u.DoubleValue>0 And x.DoubleValue>0 And t.DoubleValue>0 AndAlso Abs(q.DoubleValue*u.DoubleValue+x.DoubleValue-t.DoubleValue)<tol Then quxt=True
   If nxt And Not nxr Then
      Dim rate As Double
      rate=Round(x.DoubleValue/n.DoubleValue)
      If rate=10 Or rate=18 Then
         r.Text=Format(x.DoubleValue/n.DoubleValue,"00")
         pfl.FormatTableCell(r)
      End If
   End If
   If nrt And Not nxt Then
      x.Text=Format(n.DoubleValue*r.DoubleValue/100,"0.00")
      afl.FormatTableCell(x)
   End If
   If rxt And Not nrt Then
      n.Text=Format(t.DoubleValue-x.DoubleValue,"0.00")
      afl.FormatTableCell(n)
   End If
   If nxr And Not nrt Then
      t.Text=Format(n.DoubleValue+x.DoubleValue,"0.00")
      afl.FormatTableCell(t)
   End If
   If quxt And Not nrt Then
       n.Text=Format(t.DoubleValue-x.DoubleValue,"0.00")
       afl.FormatTableCell(n)
   End If
End Sub
```

## Amount formatter for - and =



```vbscript
Private Sub SL_Table_Rows_LocateAlternatives(ByVal pXDoc As CASCADELib.CscXDocument, ByVal pLocator As CASCADELib.CscXDocField)
   Dim lineIndex As Long, words As CscXDocWords, word As CscXDocWord, c As Long,row As CscXDocTableRow,match As Boolean
   Dim Table As CscXDocTable
   Dim tl As CscTableLocLib.CscTableLocator, MasterCells As CscXDocTableCells, cell As CscXDocTableCell
   Dim l As Long, w As Long, t As Long, h As Long, x As New CscXDocument
   x.Load(pXDoc.FileName)
   pXDoc.Fields.ItemByName("StartTime").Text=CStr(Timer)
   Set MasterCells=x.Fields.ItemByName("Table").Table.Rows(0).Cells
   Open "c:\temp\table.txt" For Output As #1
   Set Table=pXDoc.Fields.ItemByName("Table").Table
   For c = 0 To Table.Columns.Count-1
      Print #1, Table.Columns(c).Name & ";" ;
   Next
   Print #1,
   For lineIndex=0 To pXDoc.TextLines.Count-1
      Set row=Table.Rows.Append()
      Set words = pXDoc.TextLines(lineIndex).Words
      c=0
      For w =0 To words.Count-1
         Set word=words(w)
         match=False
         While c<MasterCells.Count
            If Object_OverlapHorizontal2D(word,MasterCells(c)) Then
               match=True
               Exit While
            End If
            c=c+1
            Print #1,";";
         Wend
         If match Then
            Print #1, " " & word.Text ;
            row.Cells(c).AddWordData(word)
         End If
         If c>=MasterCells.Count Then Exit For
      Next
      Print #1,
   Next
   Close #1
End Sub

Public Function Object_OverlapHorizontal2D( a As Object, b As Object,Optional offset As Long=0) As Double
   Return Max((Min(a.Left+a.Width,b.Left+b.Width+offset)-Max(a.Left,b.Left+offset)),0)>0
End Function

Public Function Max(a,b)
   Return IIf(a>b,a,b)
End Function

Public Function Min(a,b)
   Return IIf(a<b,a,b)
End Function
```
## Locating INN & KPP
The following database contains INN anchors. Make it a fuzzy database locator with substituion values
```
КПП покупателя;buyer
КПП продавца;vendor
Идентификационный номер покупателя;buyer 
Идентификационный номер продавца;vendor
```
Create a multifield Script locator that finds the INN/KPP after these anchor words. It has two subfields **VendorINNKPP** and **BuyerINNKPP**. they will be split later
```vbscript
Private Sub SL_INNKPPfromAnchors_LocateAlternatives(ByVal pXDoc As CASCADELib.CscXDocument, ByVal pLocator As CASCADELib.CscXDocField)
   Dim Anchors As CscXDocFieldAlternatives
   Set Anchors=pXDoc.Locators.ItemByName("DB_INNKPPAnchors").Alternatives
   Dim Buyer As Long, Vendor As Long,A As Long,I As Long,W As Long,Digits As Long
   Dim INNKPP As CscXDocWords
   Dim Number As Boolean
   Buyer=-1
   Vendor=-1
   For A = 0 To Anchors.Count-1
      Select Case Anchors(A).SubFields(1).Text
      Case "buyer"
         If Buyer=-1 Then Buyer=A
      Case "vendor"
         If Vendor=-1 Then Vendor=A
      End Select
   Next
   With pLocator.Alternatives.Create
      .Confidence=1
      .SubFields.Create("VendorINNKPP")
      .SubFields.Create("BuyerINNKPP")
      If Vendor>-1 Then
         Set INNKPP =XDocument_GetNextPhrase(pXDoc,Anchors(Vendor).SubFields(0),400) ' 400 pixels max gap
         Number = False
         For W = 0 To INNKPP.Count-1
            If Not Number AndAlso String_CountDigits(INNKPP(W).Text)/Len(INNKPP(W).Text)>0.5 Then Number=True
            If Number Then .SubFields(0).Words.Append(INNKPP(W))
         Next
      End If
      If Buyer>-1 Then
         Set INNKPP =XDocument_GetNextPhrase(pXDoc,Anchors(Buyer).SubFields(0),400)
         Number = False
         For W = 0 To INNKPP.Count-1
            If Not Number AndAlso String_CountDigits(INNKPP(W).Text)/Len(INNKPP(W).Text)>0.5 Then Number=True
            If Number Then .SubFields(1).Words.Append(INNKPP(W))
         Next
      End If
      For W = 0 To 1
         If Len(.SubFields(W).Text)>5 Then .SubFields(W).Confidence=1
      Next
   End With
End Sub

Private Function String_CountDigits(A As String) As Integer
   'Returns the number of digits in a word
   Dim R As Long, C As Long
   For R = 1 To Len(A)
      Select Case AscW(Mid(A, R, 1))
      Case &H30 To &H39
         C = C + 1
      End Select
   Next
   String_CountDigits = C
End Function

Public Function XDocument_GetNextPhrase(ByVal pXDoc As CASCADELib.CscXDocument,Subfield As CscXDocSubField,Pixels As Long) As CscXDocWords
   'returns the words following the region subfield that are within so many pixels
   Dim Result As CscXDocField
   Dim Phrase As CscXDocWords, Anchor As CscXDocWords
   Dim L As Long, X As Long,W As Long
   Dim word As CscXDocWord
   Set Result= New CscXDocField
   Set Phrase=Result.Words
   With Subfield
      Set Anchor=pXDoc.GetWordsInRect(.PageIndex,.Left,.Top+.Height/2,.Width,2)
      If Anchor.Count=0 Then Return Nothing
      L=Anchor(0).LineIndex
      X= Anchor(Anchor.Count-1).Left+Anchor(Anchor.Count-1).Width
      For W = Anchor(Anchor.Count-1).IndexInTextLine+1  To pXDoc.TextLines(L).Words.Count-1
         Set word=pXDoc.TextLines(L).Words(W)
         If word.Left-X>Pixels And Phrase.Count>0 Then Exit For 'gap in line too big
         Phrase.Append(word)
         X=word.Left+word.Width
      Next
   End With
   Return Phrase
End Function
```
## Split INN and KPP
```vbscript
Private Sub splitfield(pXDoc As CscXDocument,innName As String, kppName As String)
   Dim inn,kpp As CscXDocField
   Set inn=pXDoc.Fields.ItemByName(innName)
   inn.Text=Trim(Replace(inn.Text," ",""))
   Set kpp=pXDoc.Fields.ItemByName(kppName)
   kpp.Text=Trim(Replace(kpp.Text," ",""))
   Dim i,r As Long
   Dim found As Boolean
   For i = 6 To Len(inn.Text)
      Select Case AscW(Mid(inn.Text,i,1))
         Case &h030 To &h039
         Case Else
            found=True
            Exit For
      End Select
   Next
   If found AndAlso i>8 AndAlso Len(inn.Text)>15 AndAlso i<Len(inn.Text) Then
         kpp.Text=Mid(inn.Text,i+1)
         r=inn.Left+inn.Width
         kpp.Left=inn.Left+inn.Width*((i+0)/Len(inn.Text))
         kpp.Width=r-kpp.Left
         inn.Width=inn.Width*(i-1)/Len(inn.Text)
         kpp.Top=inn.Top
         kpp.Height=inn.Height
         kpp.PageIndex=inn.PageIndex
         inn.Text=Left(inn.Text,i-1)
         kpp.Confidence=inn.Confidence
   End If
End Sub
```
## INN Checksum Algorithm
This script is an INN validation rule - it checks that the checksum is valid. Add it to a **Script Validation Method**.
```vbscript
Private Sub INNChecksum_Validate(ByVal pValItem As CASCADELib.ICscXDocValidationItem, ByRef ErrDescription As String, ByRef ValidField As Boolean)
   Dim inn As String
   Const INNweights10 = "2,4,10,3,5,9,4,6,8,0"
   Const INNweights11 = "2,4,10,3,5,9,4,6,8,0" 'todo
   Const INNweights12 = "2,4,10,3,5,9,4,6,8,0" 'todo
   Dim weights10() As String
   weights10=Split(INNweights10,",")
   inn=pValItem.Text
   Dim r,x,sum,checksum As Integer
   Dim ch As String
   sum=0
   Select Case Len(inn)
      Case 10
         For r = 1 To 9
            ch=Mid(inn,r,1)
            If InStr(ch,"0123456789")<0 Then
               ValidField = False
               ErrDescription = "INN must be 10 or 12 digits"
               Exit Sub
            End If
            sum=sum+Val(ch)*Val(weights10(r-1))
         Next
         checksum=(sum Mod 11) Mod 10
         If checksum=Val(Mid(inn,r,10)) Then
            ValidField=True
         Else
            ValidField = False
            ErrDescription = "invalid INN checksum"
         End If
   Case 12
   'TODO
      Case Else
         ValidField = False
         ErrDescription = "INN must be 10 or 12 digits"
   End Select
End Sub
```
## Quick-Correct of Numerical fields.
Press "?" in a numerical field to quickly correct it with a single key stroke buy calculating it from two other vlaues. This is quicker than correcting an OCR error.
```vbscript
Private Sub ValidationForm_AfterFieldChanged(ByVal pXDoc As CASCADELib.CscXDocument, ByVal pField As CASCADELib.CscXDocField)
   If InStr(pField.Text,"?")=0 Then Exit Sub
   Dim afl As ICscFieldFormatter
   Set afl=Project.FieldFormatters.ItemByName(Project.DefaultAmountFormatter)
   Dim n,x,t As CscXDocField
   Set n = pXDoc.Fields.ItemByName("NetAmount1")
   Set x = pXDoc.Fields.ItemByName("TaxAmount1")
   Set t = pXDoc.Fields.ItemByName("Total")
   afl.FormatField(n)
   afl.FormatField(x)
   afl.FormatField(t)
   Select Case pField.Name
   Case "NetAmount1"
      If x.DoubleValue>0 And t.DoubleValue>0 Then n.Text=Replace(Format(t.DoubleValue-x.DoubleValue,"0.00"),".",",")
   Case "TaxAmount1"
      If n.DoubleValue>0 And t.DoubleValue>0 Then x.Text=Replace(Format(t.DoubleValue-n.DoubleValue,"0.00"),".",",")
   Case "Total"
      If n.DoubleValue>0 And x.DoubleValue>0 Then t.Text=Replace(Format(n.DoubleValue+x.DoubleValue,"0.00"),".",",")
   End Select
End Sub
```

## Check that the net, tax and total under the table actually match the sum of the table columns
```vbscript
Private Sub CheckTaxAndTotal_Validate(ByVal ValItems As CASCADELib.CscXDocValidationItems, ByVal pXDoc As CASCADELib.CscXDocument, ByRef ErrDescription As String, ByRef ValidField As Boolean)
   Dim oTax As ICscXDocValidationItem
   Dim oTot As ICscXDocValidationItem

   'you have to assign an amount formatter for each field where you want to use the .DoubleValue property
   Set oTax = ValItems.Item("Tax")
   If oTax.DoubleFormatted = False Then
      ValidField = False
      ErrDescription = oTax.Text & " is not formatted"
      Exit Sub
   End If
   Set oTot = ValItems.Item("Total")
   If oTot.DoubleFormatted = False Then
      ValidField = False
      ErrDescription = oTot.Text & " is not formatted"
      Exit Sub
   End If

   Dim sumNet, sumTax, sumTot As Double
   Dim table As CscXDocTable
   Set table=pXDoc.Fields.ItemByName("Table").Table
   If table.Rows.Count=0 Then
      ValidField=True
      Exit Sub
   End If
   Dim daf As ICscFieldFormatter
   Set daf=Project.FieldFormatters.ItemByName(Project.DefaultAmountFormatter)
   Table_SumColumn(table,table.Columns.ItemByName("Net Amount").IndexInTable,daf,sumNet)
   Table_SumColumn(table,table.Columns.ItemByName("Tax Amount").IndexInTable,daf,sumTax)
   Table_SumColumn(table,table.Columns.ItemByName("Total Price").IndexInTable,daf,sumTot)

   If Abs(sumTax-oTax.DoubleValue)>TOLERANCE Then
      ValidField=False
      ErrDescription="Table Tax " & Format(sumTax,"0.00") & " ≠ " & oTax.Text & " Total Tax"
      Exit Sub
   End If
   If Abs(sumTot-oTot.DoubleValue)>TOLERANCE Then
      ValidField=False
      ErrDescription="Table Total " & Format(sumTot,"0.00") & " ≠ " & oTot.Text & " Total"
      Exit Sub
   End If
   If sumNet>0 And Abs(sumNet+oTax.DoubleValue-oTot.DoubleValue)>TOLERANCE Then
      ValidField=False
      ErrDescription="Table Net + Table Tax = " & Format(sumNet,"0.00") & " + " & oTax.Text & " = " & Format(sumTot+oTax.DoubleValue,"0.00") & " ≠ " & oTot.Text & " Total"
      Exit Sub
   End If
   pXDoc.Fields.ItemByName("NetAmount1").Text=Replace(Format(oTot.DoubleValue-oTax.DoubleValue,"0.00"),".",",")
   ValidField=True
End Sub
```
## Useful functions
```vbscript
Private Function Table_SumColumn(table As CscXDocTable, colID As Integer,amountFormatter As ICscFieldFormatter,ByRef sum As Double) As Boolean
   'Sums a column in a database and returns false if any cell is invalid
   Dim r As Integer
   Dim cell As CscXDocTableCell
   For r = 0 To table.Rows.Count-1
      Set cell= table.Rows(r).Cells(colID)
      amountFormatter.FormatTableCell(cell)
      If Not cell.DoubleFormatted Then Return False
      sum=sum+cell.DoubleValue
   Next
   Return True
End Function

Private Sub AisB_Validate(ByVal ValItems As CASCADELib.CscXDocValidationItems, ByVal pXDoc As CASCADELib.CscXDocument, ByRef ErrDescription As String, ByRef ValidField As Boolean)
   Dim oA As ICscXDocValidationItem
   Dim oB As ICscXDocValidationItem

   'you have to assign an amount formatter for each field where you want to use the .DoubleValue property
   Set oA = ValItems.Item("A")
   If oA.DoubleFormatted = False Then
      ValidField = False
      ErrDescription = oA.Text & " not formatted"
      Exit Sub
   End If
   Set oB = ValItems.Item("B")
   If oB.DoubleFormatted = False Then
      ValidField = False
      ErrDescription = oB.Text & " not formatted"
      Exit Sub
   End If

   ' enter your own validation rule here
   ' Due to rounding of floating point numbers, it is recommended to compare double numbers as follows,
   ' using e.g. "abs(a + b - c) < 0.01" instead of "a + b = c"
   If (Abs(oA.DoubleValue - oB.DoubleValue) < 0.01) Then
      ValidField = True
   Else
      ValidField = False
      ErrDescription = "Table " & oA.Text & " ≠ " & oB.Text
   End If
End Sub
```
## Format Invoice Number
```vbscrpt
Private Sub InvoiceNumber_FormatField(ByVal FieldText As String, FormattedText As String, ErrDescription As String, ValidFormat As Boolean)
   If Len(FieldText) = 0 Then
      ValidFormat = False
      ErrDescription = "Invoice Number must not be empty"
   Else
      ' remove special characters "-/." from string
      FormattedText = Replace(FieldText, "от", "")
      FormattedText = Replace(FormattedText, "№", "")
      FormattedText = Replace(FormattedText, " ", "")
      ValidFormat = True
   End If
End Sub
```
## Spell Check Country Names
Load the country names into a database locator, and put the script into to a script field formatter called **CountryNameFormatter**
```
Австралия
Австрия
Азербайджан
Акротири
Албания
Алжир
Американское Самоа
Ангилья
Ангола
Андорра
Антарктида
Антигуа и Барбуда
Аргентина
Армения
Аруба
Афганистан
Ашмор и Картье острова
Багамские острова,
Бангладеш
Барбадос
Бассас-да-Индия
Бахрейн
Беларусь
Белиз
Бельгия
Бенин
Берег Слоновой Кости
Бермудские острова
Бирма
Болгария
Боливия
Босния и Герцеговина
Ботсвана
Бразилия
Британская территория Индийского океана
Британские Виргинские острова
Бруней
Буве
Буркина-Фасо
Бурунди
Бутан
Вануату
Великобритания
Венгрия
Венесуэла
Виргинские о-ва
Воссоединение
Вьетнам
Габон
Гайана
Гаити
Гамбии
Гана
Гваделупа
Гватемала
Гвинея
Гвинея-Бисау
Германия
Гибралтар
Гондурас
Гонконг
Гренада
Гренландия
Греция
Грузия
Гуам
Дания
Декелия
Джерси
Джибути
Доминика
Доминиканская Республика
Европа остров
Египет
Замбия
Западная Сахара
Западный берег реки Иордан
Зимбабве
Йемен
Израиль
Индия
Индонезия
Иордания
Ирак
Иран
Ирландия
Исландия
Испания
Италия
Кабо-Верде
Казахстан
Каймановы острова
Камбоджа
Камерун
Канада
Катар
Кения
Кипр
Киргизия
Кирибати
Китай
Кокосовые (Килинг) острова
Колумбия
Коморские острова
Конго, Демократическая Республика
Корея, Северный
Коста-Рика
Куба
Кувейт
Лаос
Латвия
Лесото
Либерия
Ливан
Ливия
Литва
Лихтенштейн
Люксембург
Маврикий
Мавритания
Мадагаскар
Майотта
Макао
Македонии
Малави
Малайзия
Мали
Мальдивы
Мальта
Марокко
Мартиника
Маршалловы острова
Мексика
Микронезия, Федеративные Штаты
Мозамбик
Молдова
Монако
Монголия
Монтсеррат
Навасса
Намибия
Науру
Непал
Нигер
Нигерия
Нидерландские Антильские острова
Нидерланды
Никарагуа
Ниуэ
Новая Зеландия
Новая Каледония
Норвегия
Объединенные Арабские Эмираты
Оман
Остров Клиппертон
Остров Мэн
Остров Норфолк
Остров Рождества
Остров Святой Елены
Остров Херд и острова Макдональд
Острова Кука
Острова Теркс и Кайкос
Островов Кораллового моря
Пакистан
Палау
Панама
Папуа-Новая Гвинея
Парагвай
Парасельские острова
Перу
Питкэрн
Польша
Португалия
Пуэрто-Рико
Республика Конго
Россия
Руанда
Румыния
Сальвадор
Самоа
Сан - Марино
Сан-Томе и Принсипи
Саудовская Аравия
Свазиленд
Святой Престол (Ватикан)
Северные Марианские острова
Сейшельские острова
Сектор Газа
Сенегал
Сен-Пьер и Микелон
Сент-Винсент и Гренадины
Сент-Китс и Невис
Сент-Люсия
Сербия и Черногория
Сингапур
Сирия
Словакия
Словения
Соединенные Штаты
Соломоновы Острова
Сомали
Спратли острова
Судан
Суринам
Сьерра-Леоне
Таджикистан
Тайвань
Таиланд
Танзания
Тимор-Лешти
Того
Токелау
Тонга
Тринидад и Тобаго
Тромлен острова
Тувалу
Тунис
Туркменистан
Турция
Уганда
Узбекистан
Украина
Уоллис и Футуна
Уругвай
Фарерские острова
Фиджи
Филиппины
Финляндия
Фолклендские (Мальвинские) острова
Франция
Французская Гвиана
Французская Полинезия
Французские Южные и Антарктические земли
Хорватия
Хуан де Нова Остров
Центрально-Африканская Республика
Чад
Чешская республика
Чили
Швейцария
Швеция
Шерстяная фуфайка
Шпицберген
Шри Ланка
Эквадор
Экваториальная Гвинея
Эритрея
Эстония
Эфиопия
ЮАР
Южная Джорджия и Южные Сандвичевы острова
Южная Корея
Ямайка
Ян-Майен
Япония
```
```vbscript
Private Sub CountryNameFormatter_FormatField(ByVal FieldText As String, FormattedText As String, ErrDescription As String, ValidFormat As Boolean)
   If Len(FieldText) = 0 Then
      ValidFormat = True
      Exit Sub
   End If
   Dim results As CscXDocField
   Set results=Database_Search("countries","",FieldText,2,0.5)
   If results.Alternatives.Count=0 Then
      ValidFormat=False
      ErrDescription="неизвестной стране"
      Exit Sub
   End If
   If results.Alternatives.Count=1 AndAlso results.Alternatives(0).Confidence>0.5 Then
      FormattedText=results.Alternatives(0).SubFields(0).Text
      ValidFormat=True
      Exit Sub
   End If
   If results.Alternatives.Count>1 AndAlso results.Alternatives(0).Confidence-results.Alternatives(1).Confidence> 0.25 Then
      FormattedText=results.Alternatives(0).SubFields(0).Text
      ValidFormat=True
      Exit Sub
   End If
   ValidFormat = False
   ErrDescription="неизвестной стране"
End Sub
```
## Units Formatting
Fuzzy match units and auto-correct them with a Script field Formatter
```vbscript
Const UNITS="БУТ,БУТЫЛК,БУТЫЛКА,ШТ,КГ,КОР,КОР.20,ВЕДРО,ПАЧ,УПАК,УПАК.8,УПАК.12,УП,БАНКА,БЛК,УПК"
Private Sub UnitsFormatter_FormatField(ByVal FieldText As String, FormattedText As String, ErrDescription As String, ValidFormat As Boolean)
   FormattedText=Replace(FieldText,".","")
   FormattedText=UCase(Replace(FormattedText,"|",""))
   If Len(FormattedText) = 0 Then
      ValidFormat = True
      Exit Sub
   End If
   Dim unit As String
   Dim bestId,bestScore,score,i As Integer
   bestScore=100
   For Each unit In Split(UNITS,",")
      score=String_LevenshteinDistance(unit,FormattedText)
      If score<bestScore Then bestScore=score:bestId=i
      i=i+1
   Next
   If bestScore<2 Then
      ValidFormat=True
      FormattedText=Split(UNITS,",")(bestId)
   Else
      ValidFormat=False
      ErrDescription="неизвестной Единица измерения"
   End If
End Sub

Private Function String_LevenshteinDistance(a As String , b As String)
   'http://en.wikipedia.org/wiki/Levenshtein_distance
   'Levenshtein distance between two strings, used for fuzzy matching
   Dim i,j,cost,d,ins,del,subs As Integer
   If Len(a) = 0 Then Return 0
   If Len(b) = 0 Then Return 0
   ReDim d(Len(a), Len(b))
   For i = 0 To Len(a)
      d(i, 0) = i
   Next
   For j = 0 To Len(b)
      d(0, j) = j
   Next
   For i = 1 To Len(a)
     For j = 1 To Len(b)
         If Mid(a, i, 1) = Mid(b, j, 1) Then cost = 0 Else cost = 1   ' cost of substitution
         del = ( d( i - 1, j ) + 1 ) ' cost of deletion
         ins = ( d( i, j - 1 ) + 1 ) ' cost of insertion
         subs = ( d( i - 1, j - 1 ) + cost ) 'cost of substition or match
         d(i,j)=Min(ins,Min(del,subs))
      Next
   Next
   Return d(Len(a), Len(b))
End Function

Private Function Max(v1 As Long, v2 As Long) As Long
   If v1 > v2 Then Return v1 Else Return v2
End Function

Private Function Min(v1 As Long, v2 As Long) As Long
   If v1 < v2 Then Return v1 Else Return v2
End Function


```
