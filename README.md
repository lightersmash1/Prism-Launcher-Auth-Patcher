# An assortment of methods to patch Prism Launcher (10.0.2+*).

***NOTE FOR ALL THAT USE THIS ON FUTURE VERSIONS THAN PRISM LAUNCHER 10.0.2**: If the first pattern is successfully patched, but one or two of the others fails, remove the section of the code that does the last two patches and follow this tutorial after doing the one patch: https://github.com/antunnitraj/Prism-Launcher-PolyMC-Offline-Bypass

AOBs involved (for those who are well off on their own or those with hex editing capabilities that extend past HxD in #3):

**49 63 85 c8 ?? ?? ?? -> b8 03 00 00 00 90 90**

**74 33 41 83 7d 20 -> eb 21 ?? ?? ?? ??**

**41 83 7c 24 60 00 7e 10 -> E9 FA 00 00 00 ?? ??**

**Use #1 for windows, #2 for windows/linux/mac, #3 for users on windows who want to use HxD and for linux/mac with a custom hex editor, #4 for windows/mac as a last resort**

# 1. Powershell

Open powershell directly or open command prompt then type "powershell". 

```powershell
function Compile-Find($s){
    $s.Split(' ') | ForEach-Object {
        if($_ -eq '??'){ -1 } else { [Convert]::ToInt32($_,16) }
    }
}

function Compile-Repl($s){
    [byte[]]($s.Split(' ') | ForEach-Object { [Convert]::ToByte($_,16) })
}

function Patch($d,$f,$r){
    if($r.Length -lt $f.Length){
        $pad = New-Object byte[] ($f.Length - $r.Length)
        $pad | ForEach-Object { $_ = 0x90 }
        $r = $r + $pad
    }

    for($i=0;$i -le $d.Length-$f.Length;$i++){
        for($j=0;$j -lt $f.Length;$j++){
            if($f[$j] -ne -1 -and $d[$i+$j] -ne $f[$j]){ break }
        }
        if($j -eq $f.Length){
            for($k=0;$k -lt $f.Length;$k++){ $d[$i+$k] = $r[$k] }
            Write-Host "[+] Patched @ 0x$('{0:X}' -f $i)"
            return
        }
    }
    throw "Pattern not found"
}

$filename = "$env:LOCALAPPDATA\Programs\PrismLauncher\prismlauncher.exe"

$data = [IO.File]::ReadAllBytes($filename)

Patch $data (Compile-Find "49 63 85 c8 ?? ?? ??")     (Compile-Repl "b8 03 00 00 00 90 90")
Patch $data (Compile-Find "74 33 41 83 7d 20")        (Compile-Repl "eb 21 00 00 00 00")
Patch $data (Compile-Find "41 83 7c 24 60 00 7e 10")  (Compile-Repl "e9 fa 00 00 00 00 00")

[IO.File]::WriteAllBytes($filename,$data
```

# 2. Python

Make a python file or enter into command prompt after typing "py". 
For linux/mac users or users who have it installed somewhere else, replace the file path in the script with the path that you have it installed.

```python
import os, sys

def patch_binary(fn, find, repl):
    d = bytearray(open(fn, "rb").read())
    f = find.split()
    r = [int(x,16) for x in repl.split()]
    r += [0x90] * (len(f) - len(r))

    for i in range(len(d) - len(f) + 1):
        if all(f[j]=="??" or d[i+j]==int(f[j],16) for j in range(len(f))):
            d[i:i+len(f)] = r
            open(fn, "wb").write(d)
            print(f"[+] Patched @ 0x{i:X}")
            return
    raise RuntimeError("Pattern not found")

filename = os.path.expandvars("%localappdata%/Programs/PrismLauncher/prismlauncher.exe")

patch_binary(filename, "49 63 85 c8 ?? ?? ??", "b8 3 0 0 0 90 90")
patch_binary(filename, "74 33 41 83 7d 20", "eb 21 0 0 0 0")
patch_binary(filename, "41 83 7c 24 60 0 7e 10", "e9 fa 0 0 0 0 0")
```

# Manual tutorials

# 3. HxD (also can be followed with a different hex editor on linux/mac) 

Open the prismlauncher exe at %localappdata%/Programs/PrismLauncher/prismlauncher.exe. MAKE A BACKUP IN CASE YOU DO THIS WRONG!!

IF AT ANY POINT IT PROMPTS SAYING THAT THE OPERATION WILL CHANGE FILESIZE, YOU'RE DOING THE STEPS WRONG, AND IF YOU PRESS YES, THE EXE WILL BREAK.

Go to Search -> Find -> Hex-values -> Enter "49 63 85 c8" -> OK/Search all. Manually type in "b8 03 00 00 00 90 90" where you were put after the search, overwriting everything in its place.

Repeat with this: 74 33 41 83 7d 20 -> eb 21
Repeat with this: 41 83 7c 24 60 00 7e 10 -> e9 fa 00 00 00

Save & Done!

# 4. Cheat Engine (warning: will not save after program is closed) (Mac-friendly)

Attach cheat engine to Prism Launcher after you open Prism Launcher. 

Press new scan (or first scan if you started with that), change value type to "Array of byte" then enter "49 63 85 c8", press first scan, then double click on the result and replace the value with "b8 03 00 00 00 90 90", then press OK.

Repeat with this: 74 33 41 83 7d 20 -> eb 21
Repeat with this: 41 83 7c 24 60 00 7e 10 -> e9 fa 00 00 00

Done!

# Explanation of how this was all found

In the LauncherController.cpp file in the source code, there's two new sections (in LaunchController::login):

` if ((m_accountToUse->accountType() != AccountType::Offline ...) {
        ...
        m_accountToUse->refresh();
    }`

This section will give an _account state_ to the Microsoft account being used (which is what m_accountToUse and accountToCheck are.) In the original bypass, it would probably make the account state be set to _Errored_.

`switch (accountToCheck->accountState()) {`

This section actually checks the state for being the correct one. The state that gives you the pass is AccountState::_Online_ (represented by the number 3 by the compiler, and yes, this will be important.)

After disassembling the exe's raw assembly, I was able to find the CPU instruction for that exact switch on Windows 11, shown below. If you want know how to find the function, and a decompiled version of it into psuedocode, get **Ghidra** for free or pay $5000 for **IDA** (or use Cheat Engine), then find the function referencing the string "Launch cancelled - account does not own Minecraft.", which is inside of that LauncherController::login function.

<img width="533" height="97" alt="Image" src="https://github.com/user-attachments/assets/c43ff44a-39bc-4ead-9f2c-c458fbd1c066" />

For those of us who don't understand, _mov_ means move (basically its storing something in RAX), RAX in this case is just where the checked accountState is being stored for the `switch (...)` statement. We want to make it think that the account state is _Online_, rather than _Errored_ or _Unchecked_. 

So, we replace that _movxsd_ with a new instruction: `mov RAX, 3` (see, this is why that AccountState was important.) This will make the function believe that every time the account state is _Online_. However, that new instruction is bigger in actual size than the original instruction. So, instead, we'll use `mov EAX, 3`. EAX is the first 4 bytes of the 8 byte register RAX, so it works this way.

I found the shortest patterns with Cheat Engine that will show to the address of that instruction, and this is the first patch.

49 63 85 c8 ?? ?? ?? -> b8 03 00 00 00 90 90

**This patch is all you would need to make Prism Launcher work again IF you still use this project's accounts.json modifier.**

The next two patches are ones that would invalidate the need for a modified accounts.json. I'll keep these short and sweet, assuming others will come forward that know how to reverse engineer will make new patches if needed.

In LaunchController::login,

`if (m_accountToUse->ownsMinecraft())
     accountToCheck = m_accountToUse;
else ...
`

This next patch will make sure that "ownsMinecraft" acts as if it's true in this if statement. 

<img width="499" height="329" alt="Image" src="https://github.com/user-attachments/assets/1c70a3e1-859b-45ce-81d6-ffd0ee08ec91" />

The last JZ is run if ownsMinecraft returns true, so I replaced the first JZ with a JMP to the address at where the last JZ went. With the patch here:

74 33 41 83 7d 20 -> eb 21 ** ** ** **

The above patch still doesn't fix everything. At the start of LaunchController::login, it runs LaunchController::decideAccount(), and the checker for a valid Minecraft account is at the top of it here (note, in the disassembly, ::decideAccount is also run at the start of ::login)

`if (accounts->count() <= 0 || !accounts->anyAccountIsValid()) {`

The part that functions as this if statement is below:

<img width="431" height="64" alt="Image" src="https://github.com/user-attachments/assets/69bd6f11-3863-4527-9dff-929e09d13c98" />

It can be replaced with a JMP to the place that the JLE will go, as that JLE goes to the next part of the function after the if statement. With the patch here:

41 83 7c 24 60 00 7e 10 -> E9 FA 00 00 00 ** **

