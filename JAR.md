# How to extract JAR files

[Nokia 3410 firmware](https://archive.org/download/munkikiscastles_1.03_1.06_fullflash/Nokia%203410%20%28NHM-2%29%20%28V04.26%29%20%28Munkiki%27s%20Castles%20v1.03%29.fls) was used for an example. However, the same steps can be performed on ANY unencrypted firmware, such as Alcatel or Siemens firmware.

The guide assumes that the JAR file exists in one piece, which is almost often the case. However, hypothetically, it can be split in its internal filesystem, especially if JAR file is not embedded from factory. However, such a case was never observed, and this would be a pretty complex behavior for low-power devices.  

*Note: Guide assumes Windows as a main operating system.*  

# Prerequisites

* Firmware/full-flash  
* [7-zip](https://www.7-zip.org/download.html) – for sanity checks.
* Trusty HEX editor with the ability to copy selections to new files. Such as WinHex
* Binary crypt-analysis tools. I recommend [CryptoChecker](https://github.com/TAbdiukov/Reverse_CryptoChecker) and [HexWalk](https://github.com/gcarmix/HexWalk/)  
	* HexWalk can also be used as a HEX editor, but I never tried it 

## Getting started with CryptoChecker
CryptoChecker is very simple to set up, but **CryptoChecker absolutely requires Windows**  
1. Download the latest version of [CryptoChecker](https://github.com/TAbdiukov/Reverse_CryptoChecker)
2. Extract the archive
3. Using command-prompt, change directory to CryptoChecker
4. Run, `cc (target_file)`
5. Observe `(target_file).cc` in the `(target_file)` directory

## Getting started with HexWalk
*Note: On Linux, you can simply use binwalk*  
*Note: These steps are relevant as of 2023, but [may change in the future](https://github.com/gcarmix/HexWalk/issues/16). If these steps are not taken, then **binwalk will load indefinitely***  

These steps are required to set up binwalk as part of HexWalk, ([more info](https://github.com/gcarmix/HexWalk/#binwalk-on-windows-os)),
1. Install latest Python
2. Download HexWalk repository (`git clone` or Code -> [Download ZIP](https://github.com/gcarmix/HexWalk/archive/refs/heads/main.zip))
3. Using command-prompt, change directory to repository's `binwalk_windows` and run `python setup.py install`
4. Download [the latest HexWalk release](https://github.com/gcarmix/HexWalk/releases) for Windows
5. Extract files to any directory.
6. Run HexWalk

### Analysing files with HexWalk

1. File -> Open -> (Select target file)
2. Analysis -> Binary Analysis
3. Observe results

# Main guide

1. Run the firmware file against several binary crypt-analysis tools. I used [CryptoChecker](https://github.com/TAbdiukov/Reverse_CryptoChecker) and [HexWalk](https://github.com/gcarmix/HexWalk/)
2. Observe preliminary detected JAR/ZIP/PK archive(s) start and end position, as well as the position of a whatever meaningful entry that follows the end position (if present).

	* CryptoChecker,
	```
	00000100                  0038097F             PK........                     Arc: Jar/Rar Zip (SFX) header signature
	00000126                  003889ED             504B0506          PK..         PKZip: End Header Signature
	00000127                  0038A8D0             http://                      U 'яhttp://mobile.club.nokia.com/' { 001E }
	```
	
	* HexWalk (binwalk)
	```
	38097f - Zip archive data, at least v1.0 to extract, name: META-INF/
	3889ed - End of Zip archive
	38b8be - Unix path: /wap.plusgsm.pl/images/1/co_nowego.wbmp
	```
	* *Note*: PKZip: End Header Signature/End of Zip archive is NOT the place where JAR file ends. It only signals that JAR file is ending!  
	
	Hence we conclude that,
	* JAR file most likely starts at 0038097F (according to both CryptoChecker and HexWalk)  
	* JAR file ends somewhere after the "End Header" at 003889ED  (according to both CryptoChecker and HexWalk)  
	* JAR file ends before any other meaningful data, for example, before 0038A8D0 (as detected by CryptoChecker. Club Nokia URL) and (00)38B8BE (as detected by HexWalk, Unix path)
	
3. Typically phone store JAD files for JAR files, where JAD files in turn store JAR file size in plaintext. Let's try our chances. Search for `MIDlet-Jar-Size`. Luckily, an entry is found at 00370102: `MIDlet-Jar-Size: 32900`
*Note: If **`MIDlet-Jar-Size`** is not present, then you will have to improvize! You can try over-assuming that JAR file up until "a whatever meaningful entry that follows the end position" or assuming that JAR file ends within 1 KB (+`0x1000`) after End Header Signature. In this case, you will end up with a padded, but functional JAR file*.  
4. Now calculate JAR file end position. Think about it, it should end at `Start position + length - 1`. -1 is required, so the resulting length matches. Hence we have, 
	* 0038097F from step 2
	* Decimal 32900 from step 3
	* = hex(0x0038097F + 32900 -1) = 0x388A02  
	*Made sure to include -1!*
5. If all is done correctly, the JAR file is from 0038097F to 0x00388A02
6. Perform sanity checks to confirm the validity of your analysis. The following checks were devised,
	* Start address data should begin with "PK"  
	* As empirically observed, the last address data should be "00" (zero-byte)  
	* As empirically observed, the final data must be an even number of zero-bytes (for example, 2, 4, 6 or 8)  
	* Logically, last address must not contain unrelated to JAR data  
	
	In our case, we have,
	* Data at 0038097F should begin with "PK - Yes
	* Value at 0x388A02 is 00 - All good
	* There are 4 zero-bytes toward the last address, starting at 0x3889FF - All good
	* 0x388A02 is after "End Header" at 003889ED  (according to both CryptoChecker and HexWalk) - Check
	* 0x388A02 is before 0038A8D0 (as detected by CryptoChecker. Club Nokia URL) - Check
	* 0x388A02 is before 0038B8BE (as detected by HexWalk, Unix path) - Check
	
	All checks have passed. Time to move on.
7. Now slice out the JAR file with a HEX editor, such as [0038097F, 0x388A02], and extract it into a .JAR file
8. Sanity check: try extracting the resulting file with 7-zip and observe that there are no warnings, such as "There are data after the end of archive".
	* Note: For some reason, 7-zip does not (always) detect "there are some data after payload data" by running a test. Therefore, perform a test extraction for a sanity check*
9. Done!

---------------------------------

***[Tim Abdiukov](https://github.com/TAbdiukov)***
