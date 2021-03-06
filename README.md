#pcodedmp.py - A VBA p-code disassembler

## Introduction

It is not widely known, but macros written in VBA (Visual Basic for Applications; the macro programming language used in Microsoft Office) exist in three different executable forms, each of which can be what is actually executed at run time, depending on the circumstances. These forms are:

- _Source code_. The original source code of the macro module is compressed and stored at the end of the module stream. This makes it relatively easy to locate and extract and most free DFIR tools for macro analysis like [oledump](https://blog.didierstevens.com/programs/oledump-py/) or [olevba](http://www.decalage.info/python/olevba) or even many professional anti-virus tools look only at this form. However, most of the time the source code is completely ignored by Office. In fact, it is possible to remove the source code (and therefore make all these tools think that there are no macros present), yet the macros will still execute without any problems. I have created a [proof of concept](http://bontchev.my.contact.bg/poc2.zip) illustrating this. Most tools will not see any macros in the documents in this archive it but if opened with the corresponding Word version (that matches the document name), it will display a message and will launch `calc.exe`. It is surprising that malware authors are not using this trick more widely.

- _P-code_. As each VBA line is entered into the VBA editor, it is immediately compiled into p-code (a pseudo code for a stack machine) and stored in a different place in the module stream. The p-code is precisely what is executed most of the time. In fact, even when you open the source of a macro module in the VBA editor, what is displayed is not the decompressed source code but the p-code decompiled into source. Only if the document is opened under a version of Office that uses a different VBA version from the one that has been used to create the document, the stored compressed source code is re-compiled into p-code and then that p-code is executed. This makes it possible to open a VBA-containing document on any version of Office that supports VBA and have the macros inside remain executable, despite the fact that the different versions of VBA use different (incompatible) p-code instructions.

- _Execodes_. When the p-code has been executed at least once, a further tokenized form of it is stored elsewhere in the document (in streams, the names of which begin with `__SRP_`, followed by a number). From there it can be executed much faster. However, the format of the execodes is extremely complex and is specific for the particular Office version (not VBA version) in which they have been created. This makes them extremely non-portable. In addition, their presence is not necessary - they can be removed and the macros will run just fine (from the p-code).

Since most of the time it is the p-code that determines what exactly a macro would do (even if neither source code, nor execodes are present), it would make sense to have a tool that can display it. This is what prompted us to create this VBA p-code disassembler.

## Installation

The script will work only in Python version 2.6 or higher. It won't work in Python 3.x, because one of the imported modules (`oletools`) does not support Python 3.x. It depends on Philippe Lagadec's package [oletools](https://github.com/decalage2/oletools), so this package has to be installed before using the script. It can be installed with the command

	pip install oletools

## Usage

The script takes as a command-line argument a list of one or more names of files or directories. If the name is an OLE2 document, it will be inspected for VBA code and the p-code of each code module will be disassembled. If the name is a directory, all the files in this directory and its subdirectories will be similarly processed. In addition to the disassembled p-code, by default the script also displays the parsed records of the `dir` stream, as well as the identifiers (variable and function names) used in the VBA modules and stored in the `_VBA_PROJECT` stream.

The script supports VBA5 (Office 97, MacOffice 98), VBA6 (Office 2000 to Office 2009) and VBA7 (Office 2010 and higher).

The script also accepts the following command-line options:

`-h`, `--help`	Displays a short explanation how to use the script and what the command-line options are.

`-v`, `--version`	Displays the version of the script.

`-n`, `--norecurse` If a name specified on the command line is a directory, process only the files in this directory; do not process the files in its subdirectories.

`-d`, `--disasmonly`	Only the p-code will be disassembled, without the parsed contents of the `dir` stream or the identifiers in the `_VBA_PROJECT` stream.

`--verbose`	The contents of the `dir` and `_VBA_PROJECT` streams is dumped in hex and ASCII form. In addition, the raw bytes of each compiled into p-code VBA line are also dumped in hex and ASCII.

For instance, using the script on one of the documents in the [proof of concept](http://bontchev.my.contact.bg/poc2.zip) mentioned above produces the following results:

	python pcodedmp.py -d Word2013.doc

	Processing file: Word2013.doc
	===============================================================================
	Module streams:
	Macros/VBA/ThisDocument - 1517 bytes
	Line #0:
	        FuncDefn (Private Sub Document_Open())
	Line #1:
	        LitStr 0x001D "This could have been a virus!"
	        Ld vbOKOnly
	        Ld vbInformation
	        Add
	        LitStr 0x0006 "Virus!"
	        ArgsCall MsgBox 0x0003
	Line #2:
	        LitStr 0x0008 "calc.exe"
	        Paren
	        ArgsCall Shell 0x0001
	Line #3:
	        EndSub

For reference, it is the result of compiling the following VBA code:

	Private Sub Document_Open()
	    MsgBox "This could have been a virus!", vbOKOnly + vbInformation, "Virus!"
	    Shell("calc.exe")
	End Sub

## To do

- Implement support of VBA3 (Excel95).

- While the script should support documents created by MacOffice, this has not been tested (and you know how well untested code usually works). This should be tested and any bugs related to it should be fixed.

- I am not an experienced Python programmer and the code is ugly. Somebody more familiar with Python than me should probably rewrite the script and make it look better.

## Change log

Version 1.0.0:	Initial version.

Version 1.1.0:	Storing the opcodes in a more efficient manner. Implemented VBA7 support. Implemented support for documents created by the 64-bit version of Office.

Version 1.2.0:	Disassembling the various declarations (`New`, `Type`, `Dim`, `ReDim`, `Sub`, `Function`, `Property`).
