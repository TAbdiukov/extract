# PPT2Flash Professional
*Adapted historical guide. ppt2flash.jp*

## Assets used

* Target .swf file
* **[CryptoChecker](https://github.com/TAbdiukov/Reverse_CryptoChecker)** and [HexWalk](https://github.com/gcarmix/HexWalk/) – [(more info)](https://github.com/TAbdiukov/extract/blob/main/JAR.md)  
* Trusty HEX editor with the ability to copy selections to new files. Such as WinHex.  
* Sothink SWF Decompiler, [(last version is from 2014).](https://es.wikipedia.org/wiki/Sothink_SWF_Decompiler)

## Trivia

PPT2Flash is a Japanese tool to convert a PowerPoint file into a SWF flash file.  

In the process of conversion, the original PPT (PowerPoint) file is not stored as-is (verbatim). Instead, the PPT file info-contents (texts, media, images, videos) are extracted, transformed, and reassembled into a new presentation format, played by the PPT2Flash engine.

## Step-by-step procedure

1. Analyze SWF with CryptoChecker and HexWalk
2. Start WinHEX, open SWF file in it.
3. Based on CryptoChecker and HexWalk outputs, methodically slice out all detected (media) info-content and save each item to a separate file.  

	*Note:* To explain the procedure, I will use  **`[info-content 1]`**, **`[info-content 2]`**, **`[info-content 3]`**, etc – to refer to the corresponding info-content detected start and end addresses.  
	* The resulting files are better to be saved according to the syntax: 
		```
		{file ID}.{presumed format}
		```  
		For example, `1.xml` . 
	* Expect to slice out some subordinary .SWF files!
	* In general, you should slice out files: from the start address of **`[info-content 1]`** to the address **`[info-content 2] - 1`**. However, exceptions apply and you need to improvise!
		- First, if according to the reports, you have the beginning of XML info-content at `nonsense characters <xml`, then of course we move the carriage to the place of the real beginning of the XML info-content
		- Secondly, similarly, if you have, for example, info-content ending at `</xml>nonsense non-XML characters`, then again, move the carriage to the real end of the info-content.
		- Third, if your **`[info-content 2]`** is clearly part of **`[info-content 1]`** (for example, if the detection is a false positive), then you need to skip **`[info-content 2]`** all the way to **`[info-content 3] - 1`** (assuming **`[info-content 3]`** is, again, not something that is clearly part of **`[info-content 1]`**. Of course, you can tweak and tune CryptoChecker (and HexWalk), but it's just easier to inprovise.
		- Fourth, if your info-content ends on the characters `"=-*##^ {file format}"`, it is wise to roll the carriage back (to the address that goes before the beginning of these characters), because these characters are the PPT2Flash's encapsulation signature of the beginning of the next info-content for .
		- Finally, expect to encounter other pitfalls, so turn on your brain and improvise!
4. Use Sothink SWF Decompiler to extract sliced out surordinate SWF files to export pictures and text, using the Sothink SWF Decompiler's <ins>Multi-File Export</ins> functionality. For some reason, Sothink SWF Decompiler crashes, if you enable exporting Sprites from large .swf files. Luckily, you won't need them, because I only need Shapes and Images. 

	Most likely the crash is due to a bug in Sothink SWF Decompiler itself, because otherwise, Sothink SWF Decompiler can successfully display Sprites in its *view mode*.

5. Enjoy exported info-content!

## PPT2Flash file structure

As a bonus, I can guess how you can likely restore the structure of the decompressed info-content back into the working executable form. I didn’t go in-depth, because I don't need it, but it can come in handy

1. First (from address 0x00000000) comes the main player engine with a size of about 250 kb.
2. Then there are XML-files `texts.xml` and `swfs.xml`. They must have UTF-16 encoding, which can easily be recognized by the factor of: [the text bytes alternating with NUL-bytes](https://stackoverflow.com/q/50070289).
3. Next come the internal service images `logo.jpg` and `photo.jpg`. 
4. Then there is one XML encoded in UTF-8. The purpose of this XML file is currently unknown (it may be a part of the previous SWF)
5. Then there are several SWF files. They are laid out sequentially in the order in which they are **first** specified inside the tag of `filename`, in the file of `swfs.xml` (ie, if the same SWF is again specified in XML, it does not matter, as duplicate entries do not count).

---------------------------------

***[Tim Abdiukov](https://github.com/TAbdiukov)***