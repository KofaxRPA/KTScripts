# Training the Text Content Locator
*(This guide is compatible with KTM, KTA, RPA, KTT and RTTI.)*

The [**Text Content Locator**](https://docshield.kofax.com/KTT/en_US/6.3.0-v15o2fs281/help/PB/ProjectBuilder/450_Extraction/TextContentLocator/c_TextContentLocator.html) is a [**Natural Language Processing**](https://en.wikipedia.org/wiki/Natural_language_processing) **(NLP)** locator for finding any tokens in a text. The other NLP locator in Kofax Transfromation is the Named Entity Locator, which is not trainable.

*   purely text based and does not use any word coordinates.
*   the only locator that ignores line-wrapping.
*   requires training from many documents. You should have many hundreds if not thousands of documents.  
*   can be trained to find any values in the text. The Text Content Locator internally [tokenizes](https://www.analyticsvidhya.com/blog/2020/05/what-is-tokenization-nlp/) the text and then extracts any values you have trained for. The Named Entity Locator looks for specific Named Entities amongst the tokens.

This guide will assume that you have an Excel file with the following format, where the exact text is in one cell and the exact values (**Amount** and **Person**) perfectly match the text. _It is VERY important that the spelling of the field values **perfectly** matches the spelling in the text, because the Text Content Locator needs to learn the context of each value. For example "600$" is directly before the word "Stone" and two words before "Rob"._

<table><tbody><tr><td>Text</td><td>Amount</td><td>Person</td></tr><tr><td>Please pay Rob Stone 600$</td><td>600$</td><td>Rob Stone</td></tr><tr><td>I want to transfer five hundred dollars to Ben Senf</td><td>five hundred dollars</td><td>Ben Senf</td></tr><tr><td>Please pay the amount of 400.30 USD to the account of Erich Kelp</td><td>400.30 USD</td><td>Erick Kelp</td></tr></tbody></table>

## Convert the Data to Text Files.

1.  Create a folder on your harddrive to put the training files.  
![](https://user-images.githubusercontent.com/47416964/123088026-88447b80-d425-11eb-8edb-73882ef6b13c.png)
1.  Enter the data into Microsoft Excel.  
![](https://user-images.githubusercontent.com/47416964/123087797-3bf93b80-d425-11eb-8108-1ca80d19d26a.png)
1.  In Microsoft Excel press **ALT-F11** to open the visual basic editor.  
![](https://user-images.githubusercontent.com/47416964/123086368-a4dfb400-d423-11eb-96f9-0cc7f5a6e867.png)
1.  Open **View/Code (F7)** to see the code window.  
![](https://user-images.githubusercontent.com/47416964/123086578-df495100-d423-11eb-9d58-a4728cd78361.png)
1.  Paste the following code. Check the starting cell "A2" and the output path "C:\\temp\\moneytransfer"  
```vba
Sub MakeTextFiles()
    Dim Cell As Range, I As Long
    I = 1
    Set Cell = ActiveSheet.Range("A2")
    While Cell.Value <> ""
       Open "C:\temp\moneytransfer\" & Format(I, "0000") & ".txt" For Output As #1
       Print #1, Cell.Value
       Close #1
       I = I + 1
       Set Cell = Cell.Offset(1, 0) ' Get next cell below
    Wend
End Sub
```
6.  Run the script by pressing **Play (F5)**.  
![](https://user-images.githubusercontent.com/47416964/123088227-c5a90900-d425-11eb-84fb-5bd8d88713a1.png)
1. Your training files have now been created.  
![image](https://user-images.githubusercontent.com/47416964/123088977-abbbf600-d426-11eb-99b2-b9d02fb99e2a.png)
## Import the Text Files into Kofax Transformation
1. Create a project in Kofax Transformation Project Builder.
2. Add a class called **moneytransfer** and add the Fields **Person** and **Amount**.
3. Add a Text Content Locator with Subfields **Person** and **Amount**.  
![image](https://user-images.githubusercontent.com/47416964/123089625-7b288c00-d427-11eb-8333-f10cd27d85a7.png)
1. Assign the Locator Subfields to the fields.  
![image](https://user-images.githubusercontent.com/47416964/123089706-94313d00-d427-11eb-9911-1cfe05955aad.png)
1. Import the Text Files by selecting **Text Files** as the Source Files.  
![image](https://user-images.githubusercontent.com/47416964/123089825-bd51cd80-d427-11eb-9965-3d620952f492.png)
1. Select all of your documents and assign them to the class **money transfer**.  *This will be needed for benchmarking later*
![image](https://user-images.githubusercontent.com/47416964/123090021-f25e2000-d427-11eb-9d04-39ebd5e268ef.png)
1. Select all of your documents and **Extract (F6)** them. *This will assign fields to all of the documents. You will see that they contain no values at 0%. This is because the Text Content Locator has found nothing.  
![image](https://user-images.githubusercontent.com/47416964/123090274-3fda8d00-d428-11eb-800f-e419c0ca1045.png)





