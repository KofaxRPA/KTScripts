
# Separation By Locator
This script will assume that every document is classified to the same class, and that a locator is used to find out where to separate the documents.  
**Trainable Document Separation** is not used. This is a customization of **Standard Document Separation**.    
![image](https://user-images.githubusercontent.com/47416964/113839226-ca97de00-978f-11eb-959c-ac4e977d2c85.png)

KT provides three events that can be used for custom [Document Separation](https://docshield.kofax.com/KTT/en_US/6.3.0-v15o2fs281/help/SCRIPT/ScriptDocumentation/c_StandardDocumentSeparation.html)
All of these events are run before the document is really split. All the Events do is provide you convenient opportunities to set CDoc.Pages(P).SplitPage to **true** or leave it at **false**.
* **Document_BeforeSeparatePages**(ByVal pXDoc As CASCADELib.CscXDocument, ByRef bSkip As Boolean) 
*This event provides you with the entire batch of pages as a single document*  
*If you set bSkip=true then **Document_SeparateCurrentPage** will not be called for each page.   **Document_SeparateCurrentPage** will always be called.*

* **Document_SeparateCurrentPage**(pXDoc As CASCADELib.CscXDocument, ByVal PageNr As Long, bSplitPage As Boolean, RemainingPages As Long)  
*This event is called for every single page and provides you with the "batch of pages" and a pointer to the current page.*  
All this script does is set pXDoc.CDoc.Pages(PageNr).SplitPage=bSplitPage and skip pages based on **RemainingPages**.  
If you set bSplitPage=true, then this page will become the first page of a new document.  
if you ignore **RemainingPages**, then the same event will be called for the next page.
* **Document_AfterSeparatePages**(ByVal pXDoc As CASCADELib.CscXDocument)  
The Document will be split AFTER **Document_AfterSeparatePages** has run. This is your last chance to change **pXDoc.CDoc.Pages(PageNr).SplitPage** for each page. This event is mainly used as an opportunity to change what  **Trainable Document Separation** or **Standard Document Separation** suggest.

# Sample Documents
The attached sample set contains 11 pages that are very simple: they contain only 1 digit on them, and 2 of them are blank. 
**1,_,1,2,3,_,4,5,6,5,5**  
These need to be separated into 6 documents  
**1-_-1, 2, 3-_, 4, 5, 6-6-6**  
Page 2 of document 1 and page 2 of document 3 are completely blank.
The most complicated thing to do is to ensure that page 3 of document 1 remains part of the document 1 and doesn't become its own document.  
We cannot use the simplistic strategy *make a new document if the locator has a different value than the previous page*. We need to look at the locator value from before the blank pages. 

# Two Strategies
Some locators find only ONE result and then stop. We need to call these locators for each page.
* database locator
* trainable group locators
* table locator
* zone locator
* Address Locator
* Vendor Locator
* Sentiment Locator  
Some locators find ALL results on all pages. We can call these locator ONCE for all pages.
* format locator
* barcode locator  

# Strategy 1. Call a separation locator once for all pages.
* Right-click on your desired class and select **Default Classification Result**. Every document will be classified to this class.    
![image](https://user-images.githubusercontent.com/47416964/113844449-c5895d80-9794-11eb-9906-2422e80d1f22.png)    * Create a class called **separation** to hold your separation locator(s)
* Disable "Valid classification result" and "Available for manual classification". This class will only be used by your script, you don't want it to be used for classification nor seen by the validation users.  
![image](https://user-images.githubusercontent.com/47416964/113843019-88709b80-9793-11eb-8ed9-ae7b95d786a4.png)* Add a field **separation** to the **separation** class.
![image](https://user-images.githubusercontent.com/47416964/114250798-313f1680-999f-11eb-8ce7-3304dd98f4a1.png)
* Load the document set and switch to **Hierarchy View**  
![image](https://user-images.githubusercontent.com/47416964/114250861-60558800-999f-11eb-9fac-923b6d86a20e.png)
* You can use the right-click **Edit Menu** to split and merge pages into the documents as needed.  
![image](https://user-images.githubusercontent.com/47416964/114250895-7d8a5680-999f-11eb-9fd2-8e98fe670cc1.png)
* Select all documents (CTRL-A) and Press F5 to classify all your documents (in KTA you can press F6 to Extract with the currently selected class). This is to ensure that each document is classified to the correct class.
* **Assign** all of your documents to the correct class as well.
* **Save** your documents  
![image](https://user-images.githubusercontent.com/47416964/114250994-cfcb7780-999f-11eb-98ef-f18460cf9b3a.png)
* Convert it to a benchmark set  
![image](https://user-images.githubusercontent.com/47416964/113845349-b2c35880-9795-11eb-9094-dcf2d7645907.png)
* You now see that the separation benchmark is available.  
![image](https://user-images.githubusercontent.com/47416964/113845046-61b36480-9795-11eb-9a93-78ee18ada45c.png)
* Run the benchmark and see that your documents are not separated.
![image](https://user-images.githubusercontent.com/47416964/113845194-89a2c800-9795-11eb-9a43-2c16ed5977bd.png)
* Add your separation locator(s) to the **separation** class. For the example documents it is a format locator looking for a single digit **\d**
* Add the script below to the Project Class. It will mark a page for splitting if the locator returns a different value than the previous page that contained a value (i.e. blank pages are just added to the the current document)
 ```vb
Private Sub Document_BeforeSeparatePages(ByVal pXDoc As CASCADELib.CscXDocument, ByRef bSkip As Boolean)
   Dim P As Long, Page As CscCDocPage, Temp As CscXDocument, SeparationValue As String, Previous As String
   For P=0 To pXDoc.CDoc.Pages.Count-1
      Set Page=pXDoc.CDoc.Pages(P)
      Set Temp=New CscXDocument
      Temp.CopyPages(pXDoc,P,1)
      Project.ClassByName("separation").Extract(Temp)
      SeparationValue=Temp.Fields.ItemByName("separation").Text
      If SeparationValue <>"" Then
         If SeparationValue<>Previous Then Page.SplitPage=True
         Previous=SeparationValue
      End If
      Set Temp=Nothing
   Next
   bSkip=True ' Don't run Document_SeparateCurrentPage. Document_AfterSeparatePages will still run
End Sub
````
* Run the separation benchmark and you will see correct separation.  
![image](https://user-images.githubusercontent.com/47416964/114251084-0bfed800-99a0-11eb-8ae9-e8f156232709.png)


# Strategy 2. Call a separation locator for each page.
*problem: This script makes page 3 of document 4 a new document*
```vb
Private Sub Document_BeforeSeparatePages(ByVal pXDoc As CASCADELib.CscXDocument, ByRef bSkip As Boolean)
   'Build an array of numbers for each page
   Dim A As Long, Alt As CscXDocFieldAlternative, Alts As CscXDocFieldAlternatives
   ReDim Numbers(pXDoc.Pages.Count-1)
   Project.RootClass.Extract(pXDoc)
   Set Alts=pXDoc.Locators.ItemByName("FL_Number").Alternatives
   For  A=0 To Alts.Count-1
      Set Alt=Alts(A)
      If Alt.Confidence>0.8 Then Numbers(Alt.PageIndex)=Alt.Text
   Next
End Sub

Private Sub Document_SeparateCurrentPage(pXDoc As CASCADELib.CscXDocument, ByVal PageNr As Long, bSplitPage As Boolean, RemainingPages As Long)
   'This keeps handing you the parent document over and over with a new pagenr. This pXDoc is not actually split, but pages are
   'copied out of it into the new "real" xdocuments
   If PageNr=0 Then ' this is to prevent the array lookup looking for -1, and we never split the first page anyway
      bSplitPage= False
   ElseIf Numbers(PageNr)="" Then ' current page has no number so part of previous document
      bSplitPage= False
   ElseIf Numbers(PageNr)=Numbers(PageNr-1) Then ' same number = same document
      bSplitPage=False
   Else
      bSplitPage=True  ' current page has a different number than previous page, so new document
   End If
End Sub
```
