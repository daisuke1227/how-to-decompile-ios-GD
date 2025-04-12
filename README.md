# Reverse Engineering Geometry Dash 2.2074 (iOS) with IDA Pro – Step-by-Step Guide

**Overview:** In this tutorial, we will reverse engineer the iOS version of *Geometry Dash 2.2074* using IDA Pro on Windows. We will leverage the Android version of the game as a reference to find corresponding functions in the iOS binary. The guide will cover setting up both binaries in IDA, navigating the disassembly, cross-referencing between ARM architectures, and handling challenges like inlined functions. The instructions are beginner-friendly yet detailed enough for intermediate reverse engineers, with clear formatting, code snippets, and illustrative examples.

## 1. Tools and Files Setup on Windows

Before we begin, ensure you have the necessary tools and game files:

- **IDA Pro:** The primary disassembler and decompiler we’ll use (legally obtained) ([Chapter 2.1: Reverse Engineering - Geode Docs](https://docs.geode-sdk.org/handbook/vol2/chap2_1/#:~:text=There%20are%20a%20lot%20of,the%20most%20common%20ones%20include)). Install it on your Windows PC.
- **Geometry Dash 2.2074 Binaries:** You need both the Android and iOS game binaries:
  - **Android APK** – Obtain the APK of Geometry Dash 2.2074 (e.g., from your Android device or backup). On Windows, you can use a tool like 7-Zip to extract the APK (it's a zip archive). Inside, find the native library (usually under `lib/armeabi-v7a/` or `lib/arm64-v8a/`). The library might be named something like `libGeometryDash.so` (or a similar name). This `.so` file is the Android native code we’ll analyze.
  - **iOS IPA** – Obtain the IPA of Geometry Dash 2.2074 (e.g., from your iOS device or App Store download). On Windows, rename the `.ipa` to `.zip` and extract it with 7-Zip. Navigate into the `Payload/Geometry Dash.app/` folder to find the **Mach-O** binary (likely just named **GeometryDash**). This is the iOS executable we’ll analyze. (Make sure you have a decrypted binary if the app is DRM-protected, since IDA cannot analyze encrypted binaries from the App Store.)

**Note:** Ensure you legally own the game on both platforms. This tutorial assumes you have obtained the binaries from your own devices for research purposes.

## 2. Loading the Binaries in IDA Pro

Next, we load each binary into IDA for analysis:

- **Android (.so) in IDA:** Launch IDA Pro and use **File → Open** to load the `libGeometryDash.so` file. IDA will detect it as an ELF binary (32-bit ARM or 64-bit ARM, depending on the device). Accept the default load options and let IDA auto-analyze. This may take a few minutes as IDA disassembles the code and tries to identify functions. Once done, you should see the **Functions** window populated with function names (likely generic names like `sub_1234` if symbols are stripped) and the assembly listing.

- **iOS (Mach-O) in IDA:** In a new IDA instance (or after closing the first), open the iOS **GeometryDash** binary. IDA will detect a Mach-O file (possibly a Fat binary containing armv7 and arm64 slices). If prompted, choose the appropriate architecture slice – for modern iOS devices this will be ARM64 (armv8). Again, proceed with analysis. After auto-analysis, you’ll see the function list and disassembly for the iOS build.

**Tip:** It’s helpful to open two instances of IDA – one for Android, one for iOS – so you can compare them side-by-side. Rename the IDA database windows or use different UI themes to avoid confusion.

## 3. IDA Pro Basics – Navigation and Views

Let’s briefly go over IDA’s interface and how to navigate the assembly, as this will be crucial in the reverse engineering process:

 ([IOS Application Security Part 26 - Patching IOS Applications using IDA Pro and Hex Fiend | Infosec](https://www.infosecinstitute.com/resources/application-security/ios-application-security-part-26-patching-ios-applications-using-ida-pro-hex-fiend/)) *IDA Pro interface with a loaded binary (iOS in this example). On the left is the Functions window (list of identified functions), and on the right is the disassembly of the selected function. The bottom shows the output/log window. In this view, the function `-[ViewController loginButtonTapped:]` (an Objective-C method) is disassembled.*  

- **Functions Window:** On the left, IDA lists all recognized functions. You can scroll or use the search bar (Ctrl+F) to find functions by name or address. For stripped binaries, these will be auto-generated names like `sub_<address>`. In our case, Geometry Dash is written in C++ (using the Cocos2d-x engine ([How 50% of GD is duplicate code : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/s80t96/how_50_of_gd_is_duplicate_code/#:~:text=This%20sounds%20dumb%20af%20at,cocos2d%20lmao%2C%20still%20kinda%20wack))), so expect many functions named `sub_xxxx` unless you or IDA apply symbols.

- **Disassembly View:** In the middle/right, IDA shows the assembly instructions for the current function. You can press **Space** to toggle between **graph view** (flowchart of basic blocks) and **text view** (linear assembly). Graph view is useful to visualize loops and branches.

- **Pseudocode View:** If you have the Hex-Rays decompiler plugin (IDA Pro includes this for many architectures), press **F5** to open the pseudocode of the current function. This decompiled C-like code can be much easier to read than raw assembly, especially for complex logic.

- **Navigation:** Double-click on a function in the Functions list to view it. Use the **X** key to see cross-references (calls to or from the function). Press **g** to *“go to address”* if you have a specific address/offset. Use **Ctrl+F** in the disassembly or pseudocode view to search for constants, strings, or other text within the file.

- **IDA Tips:** 
  - Rename functions (press **N**) or variables to keep track of what you discover (e.g., rename `sub_1234` to `PlayerJump` once you identify it).
  - Add comments (;) to note down insights.
  - Use the **Imports/Exports** tabs to see imported API calls or exported symbols (if any). On Android, you might see imports for libcocos2d, etc., which can hint at usage of the engine’s functions.

Understanding the IDA interface will help you effectively scour the binaries for equivalent functions.

## 4. Cross-Platform Similarities – Android vs. iOS Binary

Geometry Dash’s core game logic is the same across Android and iOS because it’s largely written in the cross-platform C++ engine (Cocos2d-x) ([How 50% of GD is duplicate code : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/s80t96/how_50_of_gd_is_duplicate_code/#:~:text=This%20sounds%20dumb%20af%20at,cocos2d%20lmao%2C%20still%20kinda%20wack)). This means **functionally identical code** is just compiled to different CPU architectures. Our goal is to use the Android version (which we’ll consider “known” after analysis) to locate the same functionality in the iOS version.

Key points about the two binaries:

- **Architecture Differences:** Android on most devices uses 32-bit ARM (ARMv7) or 64-bit ARM (ARMv8-A). iOS on modern devices uses 64-bit ARM (AArch64). The code does the same things, but registers and instruction syntax differ. For example:
  - An Android ARMv7 build will use registers `R0, R1, R2...` for integers and `S0, S1...` for floats, etc.
  - The iOS ARM64 build uses `X0, X1...` (or `W0, W1...` for 32-bit parts of those registers) and `S0, D0` for floating-point.
- **Similar Constants & Strings:** *Literal constants* (numbers, strings) embedded in the code are often identical across platforms. A unique float or a specific text string found in the Android `.so` will likely appear in the iOS binary as well, though possibly at a different address or in a different form. We can use these as search anchors.
- **Function Order and Offsets:** The memory layout (and thus offsets) will differ. Don’t expect function `sub_123456` in Android to be at the same offset in iOS. Instead, rely on patterns and content of the functions, not their numeric address. The community typically documents “offsets” per platform/version separately ([GitHub - geode-sdk/bindings: Addresses & function signatures for Geometry Dash](https://github.com/geode-sdk/bindings#:~:text=We%20are%20mainly%20only%20looking,the%20appropriate%20subfolder%20under%20bindings)), so we need to discover the iOS offsets by analysis, guided by Android.
- **Engine Code vs Game Code:** Both binaries likely contain a mix of engine (Cocos2d) code and game-specific code. Engine functions (e.g., rendering, scene management) will be present on both. Knowing which functions are engine vs game can help (engine functions might be identical or even have symbols). Game-specific ones (like the player’s behavior) are what we want to map across.

By understanding these similarities and differences, we can confidently use findings in one binary to inform our search in the other.

## 5. Using the Android Version as a Reference Point

We will start by analyzing the Android binary to identify some key functions and values, then find their counterparts in the iOS binary. Here’s how to approach it:

**5.1 Identify Known Features in Android:** Think of a specific game function or feature that you want to find (for example, the player’s jump function). For guidance, we can use community knowledge or observable game behavior:
- The Geometry Dash community has reverse-engineered many values. For instance, the gravity and jump velocity for the player cube are known to be roughly `94.04` (gravity) and `20.38` (jump velocity) in game units ([Find gd physics project : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/1evwq8x/find_gd_physics_project/#:~:text=g%20is%20gravity%2C%20p%20is,robot%2C%20you%20can%20round%20em)). These are unique float constants we might search for.
- Also, consider unique strings or patterns. Perhaps a particular text (like a level name or a debug message) exists in the binary that could be near relevant code. (If the game had debug logs or error messages, those could be clues, though GD might not have obvious debug strings.)

**5.2 Search for Unique Constants or Strings:** In IDA’s Android database:
- Use **Search → Immediate value** (or press `Alt+B` for binary search) to look for known constants. For example, search for the 32-bit float value `20.3755` (the cube’s jump velocity) in the `.rodata` or code sections. You can search by the hex representation of the float. The IEEE-754 hex for 20.3755 (approx) is `0x41A30000` (in little-endian bytes this would be `00 00 A3 41`). Similarly, gravity 94.04 corresponds to hex `0x42BC14E6` (`E6 14 BC 42` little-endian).
- If using IDA’s text search, you could also search for these values as text in the decompiler output if the decompiler shows them, or use the built-in **Floating constant** search in IDA if available.

When you find a reference to such a constant in the Android binary, examine it in context. For example, you might find an area in a function that loads `20.38` into a register and then stores it into a player object’s field or uses it in a calculation. That is a strong sign you’ve found the jump implementation. 

**Example (Android ARMv7 assembly snippet):**  
Suppose IDA finds the bytes corresponding to `20.38f`. The disassembly might look like: 

```asm
LDR     R0, =0x41A30000        ; Load pointer to float 20.38
VLDR    S16, [R0]              ; Load 20.38 into floating-point S16 register
STR     S16, [R4,#0x67C]       ; Store it into an object field (e.g., player Y velocity)
MOV     R0, R4
BL      sub_123456             ; Call some function (maybe play sound)
```

In this hypothetical snippet, `0x67C` could be the offset of the player’s Y-velocity in the player object. The code loads 20.38 into that field, which is likely the jump impulse. We also see a function call after setting the velocity, which could be playing the jump sound or triggering an animation. The presence of these steps (setting velocity, making the player leave ground, playing sound) together indicates *this is the jump function* or part of it.

**5.3 Confirm the Function on Android:** Once you find a candidate function:
- Look at the decompiled pseudocode (F5) for clarity. It might show something like `this->yVelocity = 20.38; this->onGround = false; playJumpSound();` which clearly indicates a jump routine.
- Check cross-references to see where this function is called. If it’s the jump function, it might be referenced in an input handler or the player update loop. Finding its references (with **X** in IDA) could tell you *when* it's invoked (e.g., when the player taps the screen).
- Rename the function in IDA (e.g., to `PlayerObject_jump()` for reference).

At this point, you have a solid understanding of what the **jump function** looks like in the Android binary (both in assembly and in logic).

## 6. Locating the Counterpart in the iOS Binary

Now that we know what to look for, we turn to the iOS Mach-O binary to find the same functionality:

**6.1 Search for the Same Patterns in iOS:** Use the clues from Android to search the iOS binary:
- Look for the **same float constants** (e.g., 20.38 or 94.04). In IDA’s iOS database, use the search for immediate value or binary sequence. Keep in mind the endianness and size: in ARM64, the constant might still appear as 4 bytes in a literal pool, or possibly as part of an 8-byte literal (ARM64 often uses paired instructions for 64-bit values, but for a float it may still embed a 4-byte pattern).
- Alternatively, search for a similar sequence of operations. In ARM64 assembly, loading a float constant and storing to a struct might look a bit different. For instance, the compiler might embed the constant in a literal data section and use an **ADR/LDR** sequence to load it. For example:

```asm
ADR     X8, #off_ABCDEF        ; X8 points to a literal pool entry with 20.38f
LDR     S0, [X8]               ; Load 20.38 into S0 register (float register)
STR     S0, [X19,#0x320]       ; Store it into an object field (perhaps yVelocity at offset 0x320)
BL      0x100123456            ; Call a function (e.g., sound)
```

The specifics (register names, offsets) will differ, but conceptually it should do the same thing: load 20.38 into the player object’s velocity and call something after.

- You can also search by **function structure**. If the Android jump function had a unique sequence of calls (like to a sound function or particle effect), find references to the analogous call in the iOS binary. For example, if you suspect a certain function plays the jump sound, find where that is called in iOS and see if the code preceding it sets a similar value.

**6.2 Verify the iOS Function:** When you think you found the candidate in iOS:
- Open the pseudocode (if available) to verify it matches the logic (e.g., does it set a velocity, change a state, etc.). Even without symbols, the structure of the code should resemble the Android version’s pseudocode, just with different variable names or offsets.
- Check if the constant values match. Seeing `20.38` or `94.04` in the decompiled output is a strong hint you’re in the right place.
- Look at cross-references in iOS as well. The jump function likely is called from the same relative context (maybe an input handler). If on Android it was called from a function we’ll call `PlayerObject::onPlayerTap()`, then in iOS the references to the jump function should be in a similarly structured function.

**6.3 Map Out the Offsets:** Once confirmed, note the address/offset of the function in the iOS binary. For example, “Player jump function is at offset 0x1004F00 in the iOS binary.” These offsets (addresses) are what modders often share as “bindings” or function addresses for each platform ([GitHub - geode-sdk/bindings: Addresses & function signatures for Geometry Dash](https://github.com/geode-sdk/bindings#:~:text=We%20are%20mainly%20only%20looking,the%20appropriate%20subfolder%20under%20bindings)). You have effectively *bound* the Android function to its iOS counterpart by matching their behavior.

By following this process, you can systematically find many functions in the iOS build using the Android build as a map. This technique leverages the common codebase: since RobTop (the developer) “coded it in cocos2d” and likely reused code ([How 50% of GD is duplicate code : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/s80t96/how_50_of_gd_is_duplicate_code/#:~:text=This%20sounds%20dumb%20af%20at,cocos2d%20lmao%2C%20still%20kinda%20wack)), most game logic functions will have an equivalent across platforms.

## 7. Inlined Functions – What They Are and How to Identify Them

One challenge you’ll encounter in reverse engineering optimized C++ code is **inlined functions**. An inline function is one that the compiler **expands at the call site** instead of generating a separate function call. This is done for small or performance-critical functions. In the context of reverse engineering:

- **No Separate Function in Disassembly:** If a function was inlined, you won’t find a distinct `sub_xxxx` for it. Instead, its code is merged into other functions. For example, if `applyGravity()` was inlined into an update loop, the series of instructions for gravity are repeated inside the update function rather than being a call.

- **Impact on Identification:** This makes it harder to locate by name or call reference, because you can’t just find “call applyGravity”. Instead, you have to recognize the *pattern* of the inlined code. As a result, **50% of GD’s code being duplicate** (as one modder observed) is likely due to many inlined snippets repeated throughout ([How 50% of GD is duplicate code : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/s80t96/how_50_of_gd_is_duplicate_code/#:~:text=I%20personally%20use%20IDA%20pro,the%20instructions%20your%20CPU%20reads)) ([How 50% of GD is duplicate code : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/s80t96/how_50_of_gd_is_duplicate_code/#:~:text=SMJSGaming)).

**How to still identify inlined logic:**

1. **Pattern Recognition:** Look for a unique sequence of operations or constants that the inlined function would use. For instance, imagine an inlined function that calculates something with a known constant or formula – that constant will appear every place the function was inlined. You can search the binary for that constant or sequence to find all occurrences. In IDA, use binary search for a sequence of bytes (instructions) that match across different locations.

2. **Consistent Structure:** If you have the decompiler, sometimes you can recognize a repeated code block in pseudocode. For example, you might notice the same 5-line block of C code appearing in multiple larger functions – a hint that block was originally a separate inline function. You can then mark it mentally or with comments. Although IDA won’t treat it as a separate function, you can still understand it as one.

3. **Use of Signatures:** Tools like BinDiff or Diaphora can help match patterns between binaries. While that’s advanced, the idea is you could create a signature from the Android binary’s version of the code (if it was a normal function there but inlined in iOS) and then scan the iOS binary for that signature. In our case, since both are likely compiled with optimization, both may have it inlined – but if one platform didn’t inline it, you’re lucky and can use that.

4. **Leveraging Constants/Strings:** Many inline functions (especially in games) use constants. For instance, if there’s an inline function for a **“collision check”** that always uses the size of the hitbox (say a constant 0.5), searching for that constant might lead you to where the collision logic sits in bigger functions.

**Example – Inline scenario:** Suppose `PlayerObject::resetStreak()` is a tiny function that just sets the streak count to 0. The compiler might inline this everywhere the game resets the streak. So instead of calls, you see instructions like `STRB WZR, [X0,#0x1F2]` (writing 0 to a byte field at offset 0x1F2) scattered in multiple places. How do you identify this as the “resetStreak” logic? You’d notice that at offset 0x1F2 in the player object, the value is set to 0 in several functions. By analyzing the Android binary or context, you might realize offset 0x1F2 corresponds to the streak count, and these few instructions are doing the inline reset. You can then label those inlined instances in comments. 

**Tips for dealing with inlining:**
- Be methodical. When you spot identical chunks of code, diff them yourself to see if they correspond to some logical function.
- Use IDA’s **search** across the program (there is a “search memory” feature) to find all instances of a sequence once you’ve identified one.
- Don’t get frustrated – inlined code increases analysis time, but it’s the same code repeated. Once you decipher it in one place, you understand it everywhere.

## 8. Practical Tips and Tricks

To wrap up, here are some practical, Windows-focused tips for using IDA and recognizing patterns between the Android and iOS builds:

- **Use Cross-References:** On both binaries, once you identify a function or an important data structure, use cross-refs (**X** shortcut) to see where it’s used. This often unravels the structure of the program. For example, finding where the `jump` function is called tells you about the input handling, and finding what calls *that* might lead to the main game loop.

- **Compare Decompiler Output:** If you have the decompiler for both ARMv7 and ARM64, open the pseudocode for the suspected matching functions side by side. Even though the variable names will differ, the logic and flow should be very similar. This is one of the quickest ways to confirm a match.

- **Heed the Engine:** Recognize which parts of the code come from Cocos2d-x (engine) vs RobTop’s own game logic. Cocos2d-x functions (like `CCNode::setPosition` or `CCLayer::init`) might be easier to identify (perhaps via known strings or methods). If you find such a function in one binary, you can find it in the other and use it as a landmark. The game’s classes (PlayerObject, LevelManager, etc.) will call these engine functions in both binaries, giving you common reference points.

- **Document Your Findings:** Keep notes of addresses and what you think they are (especially the offsets for important functions in iOS). This becomes your personal “binding” list. For instance: *Android `sub_98ABC` = iOS `sub_100123ABC` = PlayerObject::jump*.

- **Utilize the Community Resources:** The GD modding community has partially documented class structures and functions (like the Geode SDK bindings) ([GitHub - geode-sdk/bindings: Addresses & function signatures for Geometry Dash](https://github.com/geode-sdk/bindings#:~:text=We%20are%20mainly%20only%20looking,the%20appropriate%20subfolder%20under%20bindings)). While the goal is to learn how to find them yourself, you can cross-check your results against known data (if available) to verify you’ve got the right function.

- **Be Patient with IDA Auto-Analysis:** Especially on the iOS binary, IDA might not perfectly discern all functions due to inlining or thumb mode switches in ARMv7 slices. Don’t hesitate to manually define functions if you spot a code pattern that IDA missed, or adjust function boundaries if needed. This can improve the decompiler output.

- **Working on Windows:** All the steps above are doable on a Windows PC with IDA. Extracting files via 7-Zip, running IDA, and comparing side-by-side should pose no OS-specific issues. If you want to debug or run the binaries, that would be a different story (iOS binaries would need a remote debugger on a device), but for static analysis, Windows is fine.

## 9. Example Recap and Next Steps

Let’s recap with the example we focused on – the player jump:

1. **Android analysis:** Found the `jump` routine by searching for the jump velocity constant (≈20.38). Identified the assembly where it sets the velocity and calls a sound effect. Confirmed via decompilation and renamed the function to `PlayerObject_jump` in IDA ([Find gd physics project : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/1evwq8x/find_gd_physics_project/#:~:text=44.29065743940%20Robot%20big%20g,84.1490138787)).

2. **iOS analysis:** Searched for the same constant and pattern. Found an ARM64 function that matched the behavior (loading 20.38 into what appears to be the player object’s velocity field). Verified with pseudocode and cross-references that it’s called during player input. Marked this address as the jump function in the iOS IDA database.

3. **Inline considerations:** Noted that a small helper function to, say, check if the player is on ground might be inlined in the jump routine. Recognized a repeated snippet of code (checking a flag and setting `m_onGround=false`) that occurs in both binaries around the jump logic, despite no separate function for it.

By following this process, you can proceed to map out other functions: player death, score handling, collision checks, etc., one by one. Always use the Android version as a guidepost – it’s like having a hint of what the code does, which makes finding it in the iOS binary much easier.

## 10. Conclusion

Reverse engineering a game like *Geometry Dash 2.2074* on iOS using IDA Pro becomes more approachable when you cross-reference the Android build. The shared codebase means many functions have twins across platforms. By systematically using IDA’s features to search for constants, compare disassembly, and recognize patterns, you can uncover the iOS function “offsets” and implement your mods or research with confidence. Remember to consider compiler optimizations like inlining, which may hide functions in plain sight but leave telltale traces.

With practice, you’ll become faster at identifying these patterns. Take it step-by-step, document everything, and soon the maze of assembly will start to make sense. Happy reversing!

**Sources:**

1. GD Modding Community Tools – IDA Pro is a popular choice for Geometry Dash reverse engineering ([Chapter 2.1: Reverse Engineering - Geode Docs](https://docs.geode-sdk.org/handbook/vol2/chap2_1/#:~:text=There%20are%20a%20lot%20of,the%20most%20common%20ones%20include)) ([How 50% of GD is duplicate code : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/s80t96/how_50_of_gd_is_duplicate_code/#:~:text=SMJSGaming)).  
2. Geometry Dash uses the Cocos2d-x engine and shares code across platforms ([How 50% of GD is duplicate code : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/s80t96/how_50_of_gd_is_duplicate_code/#:~:text=This%20sounds%20dumb%20af%20at,cocos2d%20lmao%2C%20still%20kinda%20wack)), enabling cross-reference techniques.  
3. Community findings on game physics values (gravity, jump velocity, etc.) provide useful search constants ([Find gd physics project : r/geometrydash](https://www.reddit.com/r/geometrydash/comments/1evwq8x/find_gd_physics_project/#:~:text=g%20is%20gravity%2C%20p%20is,robot%2C%20you%20can%20round%20em)).  
4. Geode SDK Bindings and community docs confirm focus on update 2.2074 and function address documentation efforts ([GitHub - geode-sdk/bindings: Addresses & function signatures for Geometry Dash](https://github.com/geode-sdk/bindings#:~:text=We%20are%20mainly%20only%20looking,the%20appropriate%20subfolder%20under%20bindings)).
