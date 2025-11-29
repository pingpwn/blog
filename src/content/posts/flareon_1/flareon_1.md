---
title: Flare-On 12
published: 2025-11-20
description: 'Flare-on 12: Round 1'
image: "./cover.jpeg"
tags: ["Reverse Engineering", "Flare-On"]
category: 'Writeups'
draft: false 
lang: 'en'
---


# Flareon 12 Writeups


# Round 1: Drill Baby Drill!

Just like last year, we have a pygame .exe and its python source. Let's play a bit!

![screen1.png](screen1.png)
We control this baby's horizontal index and drill out holes. Looks like our goal is to reach the teddy bear, and avoid drilling into a boulder. 
Let's take a look at the source code. 
We see there are 5 levels:

```Python
LevelNames = [
    'California',
    'Ohio',
    'Death Valley',
    'Mexico',
    'The Grand Canyon'
]
```


On the DrillBaby class we find these two functions:


```Python
def hitBoulder(self):
    global boulder_layout
    boulder_level = boulder_layout[self.x]
    return boulder_level == self.drill.drill_level
    
def hitBear(self):
    return self.drill.drill_level == max_drill_level
```

Looks like the way we detect boulders is through this global boulder_layout variable, and the way we detect bears is simply making it to the maximum depth without hitting a boulder. 
Let's take a look at main():

```Python
def main():
    global background_tiles  
    global player
    global LevelNames
    global boulder_layout
    victory_mode = False
    bear_mode = False
    next_level_mode = False
    boulder_mode = False
    bear_sum = 1
    running = True
    current_level = 0
    flag_text = None

    random.shuffle(LevelNames)

    while running:
        background_tiles = BuildBackground()
        player = DrillBaby(7, 2, max_drill_level)
        boulder_layout = []
        for i in range(0, tiles_width):
            if (i != len(LevelNames[current_level])):
                boulder_layout.append(random.randint(2, max_drill_level))
            else:
                boulder_layout.append(-1)
...
 if player.hitBear():
                player.drill.retract()
                bear_sum *= player.x
                bear_mode = True

            if bear_mode:
                screen.blit(bearimage, (player.rect.x, screen_height - tile_size))
                if current_level == len(LevelNames) - 1 and not victory_mode:
                    victory_mode = True
                    flag_text = GenerateFlagText(bear_sum)
                    print("Your Flag: " + flag_text)
```


So this is how it works: 
1. The LevelNames list is shuffled
2. For horizontal index equal to the  length of the level name (eg 4 for Ohio) the "depth" of the boulder is set to -1 (meaning there is no boulder so we win at that index)
3. For every other position the boulder is set at a random depth
4. Assuming the player finishes all levels, we use the sum of all winning positions to decrypt and print the flag!

Now, the GenerateFlagText is a very simple function to reverse and we can easily calculate the correct sum. However, during the competition I didn't even take a look at GenerateFlagText(). Instead, I did something much simpler and "dirtier" ;)

Once I realized that a boulder index of -1 practically corresponds to the bear, I patched the code to print it out like so: 

```Python
for i in range(0, tiles_width):
            if (i != len(LevelNames[current_level])):
                boulder_layout.append(random.randint(2, max_drill_level))
            else:
                print(f"FOUND IT -> {i}")
                boulder_layout.append(-1)
```
and I just played the game!

![[Pasted image 20251128122333.png]]

![[Pasted image 20251128122347.png]]



# Round 2: Project Chimera

Now, this is where I personally wasted a lot of time because of a very simple but important knowledge gap of mine: Python versions matter A LOT when it comes to  python bytecode. Let me show you why.

All we're given in this challenge is a single python file named project_chimera.py. What? More python source code? Where is the actual reversing? Let's take a look inside:


```Python
import zlib
import marshal

# These are my encrypted instructions for the Sequencer.
encrypted_sequencer_data = b'x\x9cm\x96K\xcf\...'

print(f"Booting up {f"Project Chimera"} from Dr. Khem's journal...")
# Activate the Genetic Sequencer. From here, the process is automated.
sequencer_code = zlib.decompress(encrypted_sequencer_data)
exec(marshal.loads(sequencer_code))
```

Ah, of course. Python bytecode. Now, I tried using dis.dis locally to disassemble this bytecode, but it only worked up to a point and then crashed (plus much of it didn't make sense). I installed the latest python version and it still crashed. I thought that there was some sort of "backwards compatibility" meaning that if I had the latest python version it should be able to disassemble earlier ones correctly (spoiler alert: it can't). However, the solution came from an unexpected source: online python interpreters! More specifically, programiz.com worked. Disassembling the bytecode using this line `dis.dis(marshal.loads(sequencer_code))` got me through the first level of the challenge:


```
0           0 RESUME                   0

  2           2 LOAD_CONST               0 (0)
              4 LOAD_CONST               1 (None)
              6 IMPORT_NAME              0 (base64)
              8 STORE_NAME               0 (base64)

  3          10 LOAD_CONST               0 (0)
             12 LOAD_CONST               1 (None)
             14 IMPORT_NAME              1 (zlib)
             16 STORE_NAME               1 (zlib)

  4          18 LOAD_CONST               0 (0)
             20 LOAD_CONST               1 (None)
             22 IMPORT_NAME              2 (marshal)
             24 STORE_NAME               2 (marshal)

  5          26 LOAD_CONST               0 (0)
             28 LOAD_CONST               1 (None)
             30 IMPORT_NAME              3 (types)
             32 STORE_NAME               3 (types)

  8          34 LOAD_CONST               2 (b'c$|e+O>7&-6`m!Rzak~llE|2<;!(^*VQn#...iHoo')
             36 STORE_NAME               4 (encoded_catalyst_strand)

 10          38 PUSH_NULL
             40 LOAD_NAME                5 (print)
             42 LOAD_CONST               3 ('--- Calibrating Genetic Sequencer ---')
             44 CALL                     1
             52 POP_TOP

 11          54 PUSH_NULL
             56 LOAD_NAME                5 (print)
             58 LOAD_CONST               4 ('Decoding catalyst DNA strand...')
             60 CALL                     1
             68 POP_TOP

 12          70 PUSH_NULL
             72 LOAD_NAME                0 (base64)
             74 LOAD_ATTR               12 (b85decode)
             94 LOAD_NAME                4 (encoded_catalyst_strand)
             96 CALL                     1
            104 STORE_NAME               7 (compressed_catalyst)

 13         106 PUSH_NULL
            108 LOAD_NAME                1 (zlib)
            110 LOAD_ATTR               16 (decompress)
            130 LOAD_NAME                7 (compressed_catalyst)
            132 CALL                     1
            140 STORE_NAME               9 (marshalled_genetic_code)

 14         142 PUSH_NULL
            144 LOAD_NAME                2 (marshal)
            146 LOAD_ATTR               20 (loads)
            166 LOAD_NAME                9 (marshalled_genetic_code)
            168 CALL                     1
            176 STORE_NAME              11 (catalyst_code_object)

 16         178 PUSH_NULL
            180 LOAD_NAME                5 (print)
            182 LOAD_CONST               5 ('Synthesizing Catalyst Serum...')
            184 CALL                     1
            192 POP_TOP

 19         194 PUSH_NULL
            196 LOAD_NAME                3 (types)
            198 LOAD_ATTR               24 (FunctionType)
            218 LOAD_NAME               11 (catalyst_code_object)
            220 PUSH_NULL
            222 LOAD_NAME               13 (globals)
            224 CALL                     0
            232 CALL                     2
            240 STORE_NAME              14 (catalyst_injection_function)

 22         242 PUSH_NULL
            244 LOAD_NAME               14 (catalyst_injection_function)
            246 CALL                     0
            254 POP_TOP
            256 RETURN_CONST             1 (None)
``` 


As we can see, basically what this does is load the b85decode module from the base64 library, decode a big base85 string and load that as the next level of marshal bytecode (decompressing first exactly like before). 
Let's do that ourselves so we can disassemble the next step of the challenge:

```Python
from base64 import b85decode
import zlib, dis, marshal

encoded_catalyst_strand = b'c$|e+O>7&-6...iHoo'
compressed_catalyst = b85decode(encoded_catalyst_strand)
marshalled_genetic_code = zlib.decompress(compressed_catalyst)

dis.dis(marshal.loads(marshalled_genetic_code))
```

Now the level 2 bytecode is much larger:


```
  0           0 RESUME                   0

  2           2 LOAD_CONST               0 (0)
              4 LOAD_CONST               1 (None)
              6 IMPORT_NAME              0 (os)
              8 STORE_NAME               0 (os)

  3          10 LOAD_CONST               0 (0)
             12 LOAD_CONST               1 (None)
             14 IMPORT_NAME              1 (sys)
             16 STORE_NAME               1 (sys)

  4          18 LOAD_CONST               0 (0)
             20 LOAD_CONST               1 (None)
             22 IMPORT_NAME              2 (emoji)
             24 STORE_NAME               2 (emoji)

  5          26 LOAD_CONST               0 (0)
             28 LOAD_CONST               1 (None)
             30 IMPORT_NAME              3 (random)
             32 STORE_NAME               3 (random)

  6          34 LOAD_CONST               0 (0)
             36 LOAD_CONST               1 (None)
             38 IMPORT_NAME              4 (asyncio)
             40 STORE_NAME               4 (asyncio)

  7          42 LOAD_CONST               0 (0)
             44 LOAD_CONST               1 (None)
             46 IMPORT_NAME              5 (cowsay)
             48 STORE_NAME               5 (cowsay)

  8          50 LOAD_CONST               0 (0)
             52 LOAD_CONST               1 (None)
             54 IMPORT_NAME              6 (pyjokes)
             56 STORE_NAME               6 (pyjokes)

  9          58 LOAD_CONST               0 (0)
             60 LOAD_CONST               1 (None)
             62 IMPORT_NAME              7 (art)
             64 STORE_NAME               7 (art)

 10          66 LOAD_CONST               0 (0)
             68 LOAD_CONST               2 (('ARC4',))
             70 IMPORT_NAME              8 (arc4)
             72 IMPORT_FROM              9 (ARC4)
             74 STORE_NAME               9 (ARC4)
             76 POP_TOP

 15          78 LOAD_CONST               3 (<code object activate_catalyst at 0x5afdecee7b20, file "<catalyst_core>", line 15>)
             80 MAKE_FUNCTION            0
             82 STORE_NAME              10 (activate_catalyst)

 54          84 PUSH_NULL
             86 LOAD_NAME                4 (asyncio)
             88 LOAD_ATTR               22 (run)
            108 PUSH_NULL
            110 LOAD_NAME               10 (activate_catalyst)
            112 CALL                     0
            120 CALL                     1
            128 POP_TOP
            130 RETURN_CONST             1 (None)

Disassembly of <code object activate_catalyst at 0x5afdecee7b20, file "<catalyst_core>", line 15>:
 15           0 RETURN_GENERATOR
              2 POP_TOP
              4 RESUME                   0

 16           6 LOAD_CONST               1 (b'm\x1b@I\x1dAoe@\x07ZF[BL\rN\n\x0cS')
              8 STORE_FAST               0 (LEAD_RESEARCHER_SIGNATURE)

 17          10 LOAD_CONST               2 (b'r2b-\r\x9e\xf2\x1fp\x185\x82\xcf\xfc\x90\x14\xf1O\xad#]\xf3\xe2\xc0L\xd0\xc1e\x0c\xea\xec\xae\x11b\xa7\x8c\xaa!\xa1\x9d\xc2\x90')
             12 STORE_FAST               1 (ENCRYPTED_CHIMERA_FORMULA)

 19          14 LOAD_GLOBAL              1 (NULL + print)
             24 LOAD_CONST               3 ('--- Catalyst Serum Injected ---')
             26 CALL                     1
             34 POP_TOP

 20          36 LOAD_GLOBAL              1 (NULL + print)
             46 LOAD_CONST               4 ("Verifying Lead Researcher's credentials via biometric scan...")
             48 CALL                     1
             56 POP_TOP

 22          58 LOAD_GLOBAL              3 (NULL + os)
             68 LOAD_ATTR                4 (getlogin)
             88 CALL                     0
             96 LOAD_ATTR                7 (NULL|self + encode)
            116 CALL                     0
            124 STORE_FAST               2 (current_user)

 25         126 LOAD_GLOBAL              9 (NULL + bytes)
            136 LOAD_CONST               5 (<code object <genexpr> at 0x78898d836430, file "<catalyst_core>", line 25>)
            138 MAKE_FUNCTION            0
            140 LOAD_GLOBAL             11 (NULL + enumerate)
            150 LOAD_FAST                2 (current_user)
            152 CALL                     1
            160 GET_ITER
            162 CALL                     0
            170 CALL                     1
            178 STORE_FAST               3 (user_signature)

 27         180 LOAD_GLOBAL             13 (NULL + asyncio)
            190 LOAD_ATTR               14 (sleep)
            210 LOAD_CONST               6 (0.01)
            212 CALL                     1
            220 GET_AWAITABLE            0
            222 LOAD_CONST               0 (None)
        >>  224 SEND                     3 (to 234)
            228 YIELD_VALUE              2
            230 RESUME                   3
            232 JUMP_BACKWARD_NO_INTERRUPT     5 (to 224)
        >>  234 END_SEND
            236 POP_TOP

 29         238 LOAD_CONST               7 ('pending')
            240 STORE_FAST               4 (status)

 30         242 LOAD_FAST                4 (status)

 31         244 LOAD_CONST               7 ('pending')
            246 COMPARE_OP              40 (==)
            250 EXTENDED_ARG             1
            252 POP_JUMP_IF_FALSE      294 (to 842)

 32         254 LOAD_FAST                3 (user_signature)
            256 LOAD_FAST                0 (LEAD_RESEARCHER_SIGNATURE)
            258 COMPARE_OP              40 (==)
            262 POP_JUMP_IF_FALSE      112 (to 488)

 33         264 LOAD_GLOBAL             17 (NULL + art)
            274 LOAD_ATTR               18 (tprint)
            294 LOAD_CONST               8 ('AUTHENTICATION   SUCCESS')
            296 LOAD_CONST               9 ('small')
            298 KW_NAMES                10 (('font',))
            300 CALL                     2
            308 POP_TOP

 34         310 LOAD_GLOBAL              1 (NULL + print)
            320 LOAD_CONST              11 ('Biometric scan MATCH. Identity confirmed as Lead Researcher.')
            322 CALL                     1
            330 POP_TOP

 35         332 LOAD_GLOBAL              1 (NULL + print)
            342 LOAD_CONST              12 ('Finalizing Project Chimera...')
            344 CALL                     1
            352 POP_TOP

 37         354 LOAD_GLOBAL             21 (NULL + ARC4)
            364 LOAD_FAST                2 (current_user)
            366 CALL                     1
            374 STORE_FAST               5 (arc4_decipher)

 38         376 LOAD_FAST                5 (arc4_decipher)
            378 LOAD_ATTR               23 (NULL|self + decrypt)
            398 LOAD_FAST                1 (ENCRYPTED_CHIMERA_FORMULA)
            400 CALL                     1
            408 LOAD_ATTR               25 (NULL|self + decode)
            428 CALL                     0
            436 STORE_FAST               6 (decrypted_formula)

 41         438 LOAD_GLOBAL             27 (NULL + cowsay)
            448 LOAD_ATTR               28 (cow)
            468 LOAD_CONST              13 ('I am alive! The secret formula is:\n')
            470 LOAD_FAST                6 (decrypted_formula)
            472 BINARY_OP                0 (+)
            476 CALL                     1
            484 POP_TOP
            486 RETURN_CONST             0 (None)

 43     >>  488 LOAD_GLOBAL             17 (NULL + art)
            498 LOAD_ATTR               18 (tprint)
            518 LOAD_CONST              14 ('AUTHENTICATION   FAILED')
            520 LOAD_CONST               9 ('small')
            522 KW_NAMES                10 (('font',))
            524 CALL                     2
            532 POP_TOP

 44         534 LOAD_GLOBAL              1 (NULL + print)
            544 LOAD_CONST              15 ('Impostor detected, my genius cannot be replicated!')
            546 CALL                     1
            554 POP_TOP

 45         556 LOAD_GLOBAL              1 (NULL + print)
            566 LOAD_CONST              16 ('The resulting specimen has developed an unexpected, and frankly useless, sense of humor.')
            568 CALL                     1
            576 POP_TOP

 47         578 LOAD_GLOBAL             31 (NULL + pyjokes)
            588 LOAD_ATTR               32 (get_joke)
            608 LOAD_CONST              17 ('en')
            610 LOAD_CONST              18 ('all')
            612 KW_NAMES                19 (('language', 'category'))
            614 CALL                     2
            622 STORE_FAST               7 (joke)

 48         624 LOAD_GLOBAL             26 (cowsay)
            634 LOAD_ATTR               34 (char_names)
            654 LOAD_CONST              20 (1)
            656 LOAD_CONST               0 (None)
            658 BINARY_SLICE
            660 STORE_FAST               8 (animals)

 49         662 LOAD_GLOBAL              1 (NULL + print)
            672 LOAD_GLOBAL             27 (NULL + cowsay)
            682 LOAD_ATTR               36 (get_output_string)
            702 LOAD_GLOBAL             39 (NULL + random)
            712 LOAD_ATTR               40 (choice)
            732 LOAD_FAST                8 (animals)
            734 CALL                     1
            742 LOAD_GLOBAL             31 (NULL + pyjokes)
            752 LOAD_ATTR               32 (get_joke)
            772 CALL                     0
            780 CALL                     2
            788 CALL                     1
            796 POP_TOP

 50         798 LOAD_GLOBAL             43 (NULL + sys)
            808 LOAD_ATTR               44 (exit)
            828 LOAD_CONST              20 (1)
            830 CALL                     1
ERROR!
            838 POP_TOP
            840 RETURN_CONST             0 (None)

 51     >>  842 NOP

 52         844 LOAD_GLOBAL              1 (NULL + print)
            854 LOAD_CONST              21 ('System error: Unknown experimental state.')
            856 CALL                     1
            864 POP_TOP
            866 RETURN_CONST             0 (None)

 27     >>  868 CLEANUP_THROW
            870 EXTENDED_ARG             1
            872 JUMP_BACKWARD          320 (to 234)
        >>  874 CALL_INTRINSIC_1         3 (INTRINSIC_STOPITERATION_ERROR)
            876 RERAISE                  1
ExceptionTable:
  4 to 226 -> 874 [0] lasti
  228 to 228 -> 868 [2]
  230 to 868 -> 874 [0] lasti

Disassembly of <code object <genexpr> at 0x78898d836430, file "<catalyst_core>", line 25>:
 25           0 RETURN_GENERATOR
              2 POP_TOP
              4 RESUME                   0
              6 LOAD_FAST                0 (.0)
        >>    8 FOR_ITER                15 (to 42)
             12 UNPACK_SEQUENCE          2
             16 STORE_FAST               1 (i)
             18 STORE_FAST               2 (c)
             20 LOAD_FAST                2 (c)
             22 LOAD_FAST                1 (i)
             24 LOAD_CONST               0 (42)
             26 BINARY_OP                0 (+)
             30 BINARY_OP               12 (^)
             34 YIELD_VALUE              1
             36 RESUME                   1
             38 POP_TOP
             40 JUMP_BACKWARD           17 (to 8)
        >>   42 END_FOR
             44 RETURN_CONST             1 (None)
        >>   46 CALL_INTRINSIC_1         3 (INTRINSIC_STOPITERATION_ERROR)
             48 RERAISE                  1
ExceptionTable:
  4 to 44 -> 46 [0] lasti
```

You can ask your favorite LLM to turn this back into python and it will do a fairly good job, producing something like this:

```Python
import os
import sys
import emoji
import random
import asyncio
import cowsay
import pyjokes
import art
from arc4 import ARC4


async def activate_catalyst():
    LEAD_RESEARCHER_SIGNATURE = b'm\x1b@I\x1dAoe@\x07ZF[BL\rN\n\x0cS'
    ENCRYPTED_CHIMERA_FORMULA = b'r2b-\r\x9e\xf2\x1fp\x185\x82\xcf\xfc\x90\x14\xf1O\xad#]\xf3\xe2\xc0L\xd0\xc1e\x0c\xea\xec\xae\x11b\xa7\x8c\xaa!\xa1\x9d\xc2\x90'

    print('--- Catalyst Serum Injected ---')
    print("Verifying Lead Researcher's credentials via biometric scan...")

    current_user = os.getlogin().encode()

    user_signature = bytes(c ^ (i + 42) for i, c in enumerate(current_user))

    await asyncio.sleep(0.01)

    status = 'pending'

    if status == 'pending':
        if user_signature == LEAD_RESEARCHER_SIGNATURE:
            art.tprint('AUTHENTICATION   SUCCESS', font='small')
            print('Biometric scan MATCH. Identity confirmed as Lead Researcher.')
            print('Finalizing Project Chimera...')

            arc4_decipher = ARC4(current_user)
            decrypted_formula = arc4_decipher.decrypt(ENCRYPTED_CHIMERA_FORMULA).decode()

            cowsay.cow('I am alive! The secret formula is:\n' + decrypted_formula)
            return
        else:
            art.tprint('AUTHENTICATION   FAILED', font='small')
            print('Impostor detected, my genius cannot be replicated!')
            print('The resulting specimen has developed an unexpected, and frankly useless, sense of humor.')

            joke = pyjokes.get_joke(language='en', category='all')
            animals = cowsay.char_names[1:]
            print(cowsay.get_output_string(random.choice(animals), pyjokes.get_joke()))
            sys.exit(1)
            return
    else:
        print('System error: Unknown experimental state.')
        return


asyncio.run(activate_catalyst())
```

Let's summarize what this does:
1. Encrypts your PC's username and checks it against a hardcoded encrypted value (very simple custom encryption: b ^ (i + 42) for each byte b  )
2. Uses your username as an RC4 key to decrypt the Chimera Formula (hopefully the flag)

We can derive the original username and decrypt the Chimera Formula easily:

```Python
from arc4 import ARC4

LEAD_RESEARCHER_SIGNATURE = b'm\x1b@I\x1dAoe@\x07ZF[BL\rN\n\x0cS'
ENCRYPTED_CHIMERA_FORMULA = b'r2b-\r\x9e\xf2\x1fp\x185\x82\xcf\xfc\x90\x14\xf1O\xad#]\xf3\xe2\xc0L\xd0\xc1e\x0c\xea\xec\xae\x11b\xa7\x8c\xaa!\xa1\x9d\xc2\x90'

lead_researcher_username = bytes(c ^ (i + 42) for i, c in enumerate(LEAD_RESEARCHER_SIGNATURE))
print(f'Lead Researcher Username: {lead_researcher_username.decode()}')

arc4_decipher = ARC4(lead_researcher_username)
decrypted_formula = arc4_decipher.decrypt(ENCRYPTED_CHIMERA_FORMULA).decode()
print(f"Flag: {decrypted_formula}")
```

Aaaaand we get our flag!

![[Pasted image 20251128131241.png]]

# Round 3: Pretty Devilish File

Now this particular challenge was kinda... *ahem*... out of the ordinary to say the least. We're given a pdf file of all things, called pretty_devilish_file.pdf. Firefox was unable to render the PDF for some reason, but Chromium worked:

![[Pasted image 20251128132446.png]]

but no flag here. After a bit of research and playing around I landed on a tool called qpdf. Running --show-encryption yielded some interesting results:

```
qpdf --show-encryption ./pretty_devilish_file.pdf

WARNING: ./pretty_devilish_file.pdf: file is damaged
WARNING: ./pretty_devilish_file.pdf: can't find startxref
WARNING: ./pretty_devilish_file.pdf: Attempting to reconstruct cross-reference table
WARNING: ./pretty_devilish_file.pdf (trailer, offset 1412): dictionary has duplicated key /Root; last occurrence overrides earlier ones
WARNING: ./pretty_devilish_file.pdf (object 7 0, offset 1402): expected endobj
WARNING: ./pretty_devilish_file.pdf (trailer, offset 1410): invalid /ID in trailer dictionary
R = 6
P = -1
User password = 
Supplied password is owner password
Supplied password is user password
extract for accessibility: allowed
extract for any purpose: allowed
print low resolution: allowed
print high resolution: allowed
modify document assembly: allowed
modify forms: allowed
modify annotations: allowed
modify other: allowed
modify anything: allowed
stream encryption method: AESv3
string encryption method: AESv3
file encryption method: AESv3
qpdf: operation succeeded with warnings
```

The tool claims this PDF uses AESv3 encryption, but we were never asked for a password when opened with Chromium (wtf ??). When I mentioned this in ChatGPT it claimed that the PDF could be encrypted with a blank password.  I thought this could be the case so I tried this command: `qpdf --decrypt --password='' pretty_devilish_file.pdf decrypted.pdf` and it worked. Let's look at the new file's strings:


```
strings decrypted.pdf 
%PDF-2.0
1 0 obj
<< /Extensions << /ADBE << /BaseVersion /2.0 /ExtensionLevel 8 >> >> /Pages 2 0 R /Type /Catalog >>
endobj
2 0 obj
<< /Count 1 /Kids [ 3 0 R ] /Type /Pages >>
endobj
3 0 obj
<< /Contents 4 0 R /MediaBox [ 0 0 612 130 ] /Parent 2 0 R /Resources 5 0 R /Type /Page >>
endobj
4 0 obj
<< /Filter /FlateDecode /Length 290 >>
stream
jhmDD
h1"_6
j='k8
re;2:*Vhd1
B540
=endstream
endobj
5 0 obj
<< /Font << / << /BaseFont /Arial /Subtype /Type1 /Type /Font >> >> >>
endobj
xref
0000000000 65535 f 
0000000015 00000 n 
0000000130 00000 n 
0000000189 00000 n 
0000000295 00000 n 
0000000656 00000 n 
trailer << /Root 1 0 R /ID [<064542074c2d50c3f8c584b8bcfbbacb><064542074c2d50c3f8c584b8bcfbbacb>] >>
startxref
%%EOF
```



xxd -r -p imagedata.hex > image.jpg


