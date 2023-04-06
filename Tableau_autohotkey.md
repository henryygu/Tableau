# Grievance 1

in the Calculated field window, when you double click a name, it will select the square brackets automatically. ![2023-04-06-06-13-07-26-tableau_gWHaGRperP](/assets/2023-04-06-06-13-07-26-tableau_gWHaGRperP.png)

But when I want to paste the name in the search bar, it adds the square brackets and the search fails.

## Autohotkey code

Win + C : edit the clipboard to remove the square brackets

```Autohotkey
#c::
	ClipSave := ClipboardAll
	Send, {Ctrl down}c{Ctrl up}
    If RegExMatch(Clipboard, "\[(.*?)\]", tableaunamecopy)
	{
	tableaunamecopy:= StrReplace(tableaunamecopy,"[","")
	tableaunamecopy:= StrReplace(tableaunamecopy,"]","")
    Clipboard := tableaunamecopy
    return
	}
return
```