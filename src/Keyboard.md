# Keyboard Bindings

We want to be able to launch windows or do some basic management with the keyboard,
since we haven't implemented any other way to do it.

We also want to be able to hit Ctrl-Alt-Backspace from any window,
not just the root window in order to quit. To do either of these, we'll have
to ensure we grab the keys we want with xproto.GrabKey, so that we get the
key event even if we're in another window. Let's start by adding a Grab Keys
section to our initialization, right after loading the keymap.

## Grabbing Keys Regardless of Focus

### "Initialize X"
```go
<<<Connect to X Server>>>
<<<Get Setup Information>>>
<<<Initialize Xinerama>>>
<<<Query Attached Screens>>>
<<<Set xroot to Root Window>>>
<<<Take WM Ownership>>>
<<<Load KeyMapping>>>
<<<Grab Keys>>>
<<<Gather All Windows>>>
```

The definition of GrabKey (and GrabKeyChecked) is

```go
func GrabKey(c *xgb.Conn, OwnerEvents bool, GrabWindow Window, Modifiers uint16, Key Keycode, PointerMode byte, KeyboardMode byte) GrabKeyCookie
```

We have the connection, we can figure out OwnerEvents even if it's poorly
documented (there's only 2 possibilities), GrabWindow is the root windows..but
keycode might be a problem. We loaded the KeySyms, not the keycodes. It's not
insurmountable, though, because we can iterate through our KeySym map to find
the index of the symbol that we're looking for, and use that as the keycode
(since we built the keysym array from the keycodes in the first place.)

But what are PointerMode and KeyboardMode?

The GoDocs don't say, but the X11 documentation defines the valid values as:

> pointer-mode, keyboard-mode: { Synchronous, Asynchronous}

The xproto GoDocs *do* have these consts defined:

```go
const (
    GrabModeSync  = 0
    GrabModeAsync = 1
)
```

The [X11 specification](https://www.x.org/releases/X11R7.7/doc/xproto/x11protocol.html#requests:GrabKeyboard) says:

> If keyboard-mode is Asynchronous, keyboard event processing continues normally.
> If the keyboard is currently frozen by this client, then processing of keyboard
> events is resumed. If keyboard-mode is Synchronous, the state of the keyboard
> (as seen by means of the protocol) appears to freeze. No further keyboard events
> are generated by the server until the grabbing client issues a releasing
> AllowEvents request or until the keyboard grab is released. Actual keyboard
> changes are not lost while the keyboard is frozen. They are simply queued for
> later processing.
>
> If pointer-mode is Asynchronous, pointer event processing is unaffected by
> activation of the grab. If pointer-mode is Synchronous, the state of the
> pointer (as seen by means of the protocol) appears to freeze. No further
> pointer events are generated by the server until the grabbing client issues
> a releasing AllowEvents request or until the keyboard grab is released.
> Actual pointer changes are not lost while the pointer is frozen. They are
> simply queued for later processing.

Async it is!

Let's start with Ctrl-Alt-Backspace. We already know the mask from our
Initialization.md

We iterate through our keymap looking for any XK_BackSpace keysym, as discussed
above.

### "Grab Keys"
```go
for i, syms := range keymap {
	keycode := xproto.Keycode(0)
	// Go through all the SymsPerKeycode looking for keysym.XK_BackSpace
	// Grab them all.

	for _, sym := range syms {
		if sym == keysym.XK_BackSpace {
			keycode = xproto.Keycode(i)
			break
		}
	}

	if err := xproto.GrabKeyChecked(
		xc,
		false,
		xroot.Root,
		xproto.ModMaskControl | xproto.ModMask1,
		keycode,
		xproto.GrabModeAsync,
		xproto.GrabModeAsync,
	).Check(); err != nil {
		log.Print(err)
	}
}
```

## Spawning a Terminal

Hooray! Ctrl-Alt-Backspace quits wherever we are! Let's add Alt-E to spawn
an xterm, because it's a habit burned into my muscle memory from pwm. Let's
add a hard-coded Alt-E to spawn an terminal, just so that I don't keep pressing
it and not being able to figure out why nothing is happening.

We'll have to add XK_e to our keysym map, with the value from the X11 keysymdef.h,
but while we're at it let's add all of the letters from Latin1, and the
standard up/down/left/right keys.

### "Known KeySym definitions" +=
```go
// Latin 1
// (ISO/IEC 8859-1 = Unicode U+0020..U+00FF)
// Byte 3 = 0
XK_space = 0x0020        // U+0020 SPACE
XK_exclam = 0x0021       // U+0021 EXCLAMATION MARK
XK_quotedbl = 0x0022     // U+0022 QUOTATION MARK
XK_numbersign = 0x0023   // U+0023 NUMBER SIGN
XK_dollar = 0x0024       // U+0024 DOLLAR SIGN
XK_percent = 0x0025      // U+0025 PERCENT SIGN
XK_ampersand = 0x0026    // U+0026 AMPERSAND
XK_apostrophe = 0x0027   // U+0027 APOSTROPHE
XK_quoteright = 0x0027   // deprecated
XK_parenleft = 0x0028    // U+0028 LEFT PARENTHESIS
XK_parenright = 0x0029   // U+0029 RIGHT PARENTHESIS
XK_asterisk = 0x002a     // U+002A ASTERISK
XK_plus = 0x002b         // U+002B PLUS SIGN
XK_comma = 0x002c        // U+002C COMMA
XK_minus = 0x002d        // U+002D HYPHEN-MINUS
XK_period = 0x002e       // U+002E FULL STOP
XK_slash = 0x002f        // U+002F SOLIDUS
XK_0 = 0x0030            // U+0030 DIGIT ZERO
XK_1 = 0x0031            // U+0031 DIGIT ONE
XK_2 = 0x0032            // U+0032 DIGIT TWO
XK_3 = 0x0033            // U+0033 DIGIT THREE
XK_4 = 0x0034            // U+0034 DIGIT FOUR
XK_5 = 0x0035            // U+0035 DIGIT FIVE
XK_6 = 0x0036            // U+0036 DIGIT SIX
XK_7 = 0x0037            // U+0037 DIGIT SEVEN
XK_8 = 0x0038            // U+0038 DIGIT EIGHT
XK_9 = 0x0039            // U+0039 DIGIT NINE
XK_colon = 0x003a        // U+003A COLON
XK_semicolon = 0x003b    // U+003B SEMICOLON
XK_less = 0x003c         // U+003C LESS-THAN SIGN
XK_equal = 0x003d        // U+003D EQUALS SIGN
XK_greater = 0x003e      // U+003E GREATER-THAN SIGN
XK_question = 0x003f     // U+003F QUESTION MARK
XK_at = 0x0040           // U+0040 COMMERCIAL AT
XK_A = 0x0041            // U+0041 LATIN CAPITAL LETTER A
XK_B = 0x0042            // U+0042 LATIN CAPITAL LETTER B
XK_C = 0x0043            // U+0043 LATIN CAPITAL LETTER C
XK_D = 0x0044            // U+0044 LATIN CAPITAL LETTER D
XK_E = 0x0045            // U+0045 LATIN CAPITAL LETTER E
XK_F = 0x0046            // U+0046 LATIN CAPITAL LETTER F
XK_G = 0x0047            // U+0047 LATIN CAPITAL LETTER G
XK_H = 0x0048            // U+0048 LATIN CAPITAL LETTER H
XK_I = 0x0049            // U+0049 LATIN CAPITAL LETTER I
XK_J = 0x004a            // U+004A LATIN CAPITAL LETTER J
XK_K = 0x004b            // U+004B LATIN CAPITAL LETTER K
XK_L = 0x004c            // U+004C LATIN CAPITAL LETTER L
XK_M = 0x004d            // U+004D LATIN CAPITAL LETTER M
XK_N = 0x004e            // U+004E LATIN CAPITAL LETTER N
XK_O = 0x004f            // U+004F LATIN CAPITAL LETTER O
XK_P = 0x0050            // U+0050 LATIN CAPITAL LETTER P
XK_Q = 0x0051            // U+0051 LATIN CAPITAL LETTER Q
XK_R = 0x0052            // U+0052 LATIN CAPITAL LETTER R
XK_S = 0x0053            // U+0053 LATIN CAPITAL LETTER S
XK_T = 0x0054            // U+0054 LATIN CAPITAL LETTER T
XK_U = 0x0055            // U+0055 LATIN CAPITAL LETTER U
XK_V = 0x0056            // U+0056 LATIN CAPITAL LETTER V
XK_W = 0x0057            // U+0057 LATIN CAPITAL LETTER W
XK_X = 0x0058            // U+0058 LATIN CAPITAL LETTER X
XK_Y = 0x0059            // U+0059 LATIN CAPITAL LETTER Y
XK_Z = 0x005a            // U+005A LATIN CAPITAL LETTER Z
XK_bracketleft = 0x005b  // U+005B LEFT SQUARE BRACKET
XK_backslash = 0x005c    // U+005C REVERSE SOLIDUS
XK_bracketright = 0x005d // U+005D RIGHT SQUARE BRACKET
XK_asciicircum = 0x005e  // U+005E CIRCUMFLEX ACCENT
XK_underscore = 0x005f   // U+005F LOW LINE
XK_grave = 0x0060        // U+0060 GRAVE ACCENT
XK_quoteleft = 0x0060    // deprecated
XK_a = 0x0061            // U+0061 LATIN SMALL LETTER A
XK_b = 0x0062            // U+0062 LATIN SMALL LETTER B
XK_c = 0x0063            // U+0063 LATIN SMALL LETTER C
XK_d = 0x0064            // U+0064 LATIN SMALL LETTER D
XK_e = 0x0065            // U+0065 LATIN SMALL LETTER E
XK_f = 0x0066            // U+0066 LATIN SMALL LETTER F
XK_g = 0x0067            // U+0067 LATIN SMALL LETTER G
XK_h = 0x0068            // U+0068 LATIN SMALL LETTER H
XK_i = 0x0069            // U+0069 LATIN SMALL LETTER I
XK_j = 0x006a            // U+006A LATIN SMALL LETTER J
XK_k = 0x006b            // U+006B LATIN SMALL LETTER K
XK_l = 0x006c            // U+006C LATIN SMALL LETTER L
XK_m = 0x006d            // U+006D LATIN SMALL LETTER M
XK_n = 0x006e            // U+006E LATIN SMALL LETTER N
XK_o = 0x006f            // U+006F LATIN SMALL LETTER O
XK_p = 0x0070            // U+0070 LATIN SMALL LETTER P
XK_q = 0x0071            // U+0071 LATIN SMALL LETTER Q
XK_r = 0x0072            // U+0072 LATIN SMALL LETTER R
XK_s = 0x0073            // U+0073 LATIN SMALL LETTER S
XK_t = 0x0074            // U+0074 LATIN SMALL LETTER T
XK_u = 0x0075            // U+0075 LATIN SMALL LETTER U
XK_v = 0x0076            // U+0076 LATIN SMALL LETTER V
XK_w = 0x0077            // U+0077 LATIN SMALL LETTER W
XK_x = 0x0078            // U+0078 LATIN SMALL LETTER X
XK_y = 0x0079            // U+0079 LATIN SMALL LETTER Y
XK_z = 0x007a            // U+007A LATIN SMALL LETTER Z
XK_braceleft = 0x007b    // U+007B LEFT CURLY BRACKET
XK_bar = 0x007c          // U+007C VERTICAL LINE
XK_braceright = 0x007d   // U+007D RIGHT CURLY BRACKET
XK_asciitilde = 0x007e   // U+007E TILDE

XK_nobreakspace = 0x00a0   // U+00A0 NO-BREAK SPACE
XK_exclamdown = 0x00a1     // U+00A1 INVERTED EXCLAMATION MARK
XK_cent = 0x00a2           // U+00A2 CENT SIGN
XK_sterling = 0x00a3       // U+00A3 POUND SIGN
XK_currency = 0x00a4       // U+00A4 CURRENCY SIGN
XK_yen = 0x00a5            // U+00A5 YEN SIGN
XK_brokenbar = 0x00a6      // U+00A6 BROKEN BAR
XK_section = 0x00a7        // U+00A7 SECTION SIGN
XK_diaeresis = 0x00a8      // U+00A8 DIAERESIS
XK_copyright = 0x00a9      // U+00A9 COPYRIGHT SIGN
XK_ordfeminine = 0x00aa    // U+00AA FEMININE ORDINAL INDICATOR
XK_guillemotleft = 0x00ab  // U+00AB LEFT-POINTING DOUBLE ANGLE QUOTATION MARK
XK_notsign = 0x00ac        // U+00AC NOT SIGN
XK_hyphen = 0x00ad         // U+00AD SOFT HYPHEN
XK_registered = 0x00ae     // U+00AE REGISTERED SIGN
XK_macron = 0x00af         // U+00AF MACRON
XK_degree = 0x00b0         // U+00B0 DEGREE SIGN
XK_plusminus = 0x00b1      // U+00B1 PLUS-MINUS SIGN
XK_twosuperior = 0x00b2    // U+00B2 SUPERSCRIPT TWO
XK_threesuperior = 0x00b3  // U+00B3 SUPERSCRIPT THREE
XK_acute = 0x00b4          // U+00B4 ACUTE ACCENT
XK_mu = 0x00b5             // U+00B5 MICRO SIGN
XK_paragraph = 0x00b6      // U+00B6 PILCROW SIGN
XK_periodcentered = 0x00b7 // U+00B7 MIDDLE DOT
XK_cedilla = 0x00b8        // U+00B8 CEDILLA
XK_onesuperior = 0x00b9    // U+00B9 SUPERSCRIPT ONE
XK_masculine = 0x00ba      // U+00BA MASCULINE ORDINAL INDICATOR
XK_guillemotright = 0x00bb // U+00BB RIGHT-POINTING DOUBLE ANGLE QUOTATION MARK
XK_onequarter = 0x00bc     // U+00BC VULGAR FRACTION ONE QUARTER
XK_onehalf = 0x00bd        // U+00BD VULGAR FRACTION ONE HALF
XK_threequarters = 0x00be  // U+00BE VULGAR FRACTION THREE QUARTERS
XK_questiondown = 0x00bf   // U+00BF INVERTED QUESTION MARK
XK_Agrave = 0x00c0         // U+00C0 LATIN CAPITAL LETTER A WITH GRAVE
XK_Aacute = 0x00c1         // U+00C1 LATIN CAPITAL LETTER A WITH ACUTE
XK_Acircumflex = 0x00c2    // U+00C2 LATIN CAPITAL LETTER A WITH CIRCUMFLEX
XK_Atilde = 0x00c3         // U+00C3 LATIN CAPITAL LETTER A WITH TILDE
XK_Adiaeresis = 0x00c4     // U+00C4 LATIN CAPITAL LETTER A WITH DIAERESIS
XK_Aring = 0x00c5          // U+00C5 LATIN CAPITAL LETTER A WITH RING ABOVE
XK_AE = 0x00c6             // U+00C6 LATIN CAPITAL LETTER AE
XK_Ccedilla = 0x00c7       // U+00C7 LATIN CAPITAL LETTER C WITH CEDILLA
XK_Egrave = 0x00c8         // U+00C8 LATIN CAPITAL LETTER E WITH GRAVE
XK_Eacute = 0x00c9         // U+00C9 LATIN CAPITAL LETTER E WITH ACUTE
XK_Ecircumflex = 0x00ca    // U+00CA LATIN CAPITAL LETTER E WITH CIRCUMFLEX
XK_Ediaeresis = 0x00cb     // U+00CB LATIN CAPITAL LETTER E WITH DIAERESIS
XK_Igrave = 0x00cc         // U+00CC LATIN CAPITAL LETTER I WITH GRAVE
XK_Iacute = 0x00cd         // U+00CD LATIN CAPITAL LETTER I WITH ACUTE
XK_Icircumflex = 0x00ce    // U+00CE LATIN CAPITAL LETTER I WITH CIRCUMFLEX
XK_Idiaeresis = 0x00cf     // U+00CF LATIN CAPITAL LETTER I WITH DIAERESIS
XK_ETH = 0x00d0            // U+00D0 LATIN CAPITAL LETTER ETH
XK_Eth = 0x00d0            // deprecated
XK_Ntilde = 0x00d1         // U+00D1 LATIN CAPITAL LETTER N WITH TILDE
XK_Ograve = 0x00d2         // U+00D2 LATIN CAPITAL LETTER O WITH GRAVE
XK_Oacute = 0x00d3         // U+00D3 LATIN CAPITAL LETTER O WITH ACUTE
XK_Ocircumflex = 0x00d4    // U+00D4 LATIN CAPITAL LETTER O WITH CIRCUMFLEX
XK_Otilde = 0x00d5         // U+00D5 LATIN CAPITAL LETTER O WITH TILDE
XK_Odiaeresis = 0x00d6     // U+00D6 LATIN CAPITAL LETTER O WITH DIAERESIS
XK_multiply = 0x00d7       // U+00D7 MULTIPLICATION SIGN
XK_Oslash = 0x00d8         // U+00D8 LATIN CAPITAL LETTER O WITH STROKE
XK_Ooblique = 0x00d8       // U+00D8 LATIN CAPITAL LETTER O WITH STROKE
XK_Ugrave = 0x00d9         // U+00D9 LATIN CAPITAL LETTER U WITH GRAVE
XK_Uacute = 0x00da         // U+00DA LATIN CAPITAL LETTER U WITH ACUTE
XK_Ucircumflex = 0x00db    // U+00DB LATIN CAPITAL LETTER U WITH CIRCUMFLEX
XK_Udiaeresis = 0x00dc     // U+00DC LATIN CAPITAL LETTER U WITH DIAERESIS
XK_Yacute = 0x00dd         // U+00DD LATIN CAPITAL LETTER Y WITH ACUTE
XK_THORN = 0x00de          // U+00DE LATIN CAPITAL LETTER THORN
XK_Thorn = 0x00de          // deprecated
XK_ssharp = 0x00df         // U+00DF LATIN SMALL LETTER SHARP S
XK_agrave = 0x00e0         // U+00E0 LATIN SMALL LETTER A WITH GRAVE
XK_aacute = 0x00e1         // U+00E1 LATIN SMALL LETTER A WITH ACUTE
XK_acircumflex = 0x00e2    // U+00E2 LATIN SMALL LETTER A WITH CIRCUMFLEX
XK_atilde = 0x00e3         // U+00E3 LATIN SMALL LETTER A WITH TILDE
XK_adiaeresis = 0x00e4     // U+00E4 LATIN SMALL LETTER A WITH DIAERESIS
XK_aring = 0x00e5          // U+00E5 LATIN SMALL LETTER A WITH RING ABOVE
XK_ae = 0x00e6             // U+00E6 LATIN SMALL LETTER AE
XK_ccedilla = 0x00e7       // U+00E7 LATIN SMALL LETTER C WITH CEDILLA
XK_egrave = 0x00e8         // U+00E8 LATIN SMALL LETTER E WITH GRAVE
XK_eacute = 0x00e9         // U+00E9 LATIN SMALL LETTER E WITH ACUTE
XK_ecircumflex = 0x00ea    // U+00EA LATIN SMALL LETTER E WITH CIRCUMFLEX
XK_ediaeresis = 0x00eb     // U+00EB LATIN SMALL LETTER E WITH DIAERESIS
XK_igrave = 0x00ec         // U+00EC LATIN SMALL LETTER I WITH GRAVE
XK_iacute = 0x00ed         // U+00ED LATIN SMALL LETTER I WITH ACUTE
XK_icircumflex = 0x00ee    // U+00EE LATIN SMALL LETTER I WITH CIRCUMFLEX
XK_idiaeresis = 0x00ef     // U+00EF LATIN SMALL LETTER I WITH DIAERESIS
XK_eth = 0x00f0            // U+00F0 LATIN SMALL LETTER ETH
XK_ntilde = 0x00f1         // U+00F1 LATIN SMALL LETTER N WITH TILDE
XK_ograve = 0x00f2         // U+00F2 LATIN SMALL LETTER O WITH GRAVE
XK_oacute = 0x00f3         // U+00F3 LATIN SMALL LETTER O WITH ACUTE
XK_ocircumflex = 0x00f4    // U+00F4 LATIN SMALL LETTER O WITH CIRCUMFLEX
XK_otilde = 0x00f5         // U+00F5 LATIN SMALL LETTER O WITH TILDE
XK_odiaeresis = 0x00f6     // U+00F6 LATIN SMALL LETTER O WITH DIAERESIS
XK_division = 0x00f7       // U+00F7 DIVISION SIGN
XK_oslash = 0x00f8         // U+00F8 LATIN SMALL LETTER O WITH STROKE
XK_ooblique = 0x00f8       // U+00F8 LATIN SMALL LETTER O WITH STROKE
XK_ugrave = 0x00f9         // U+00F9 LATIN SMALL LETTER U WITH GRAVE
XK_uacute = 0x00fa         // U+00FA LATIN SMALL LETTER U WITH ACUTE
XK_ucircumflex = 0x00fb    // U+00FB LATIN SMALL LETTER U WITH CIRCUMFLEX
XK_udiaeresis = 0x00fc     // U+00FC LATIN SMALL LETTER U WITH DIAERESIS
XK_yacute = 0x00fd         // U+00FD LATIN SMALL LETTER Y WITH ACUTE
XK_thorn = 0x00fe          // U+00FE LATIN SMALL LETTER THORN
XK_ydiaeresis = 0x00ff     // U+00FF LATIN SMALL LETTER Y WITH DIAERESIS

// Cursor control & motion
XK_Home = 0xff50
XK_Left = 0xff51  // Move left, left arrow
XK_Up = 0xff52    // Move up, up arrow
XK_Right = 0xff53 // Move right, right arrow
XK_Down = 0xff54  // Move down, down arrow
XK_Prior = 0xff55 // Prior, previous
XK_Page_Up = 0xff55
XK_Next = 0xff56 // Next
XK_Page_Down = 0xff56
XK_End = 0xff57   // EOL
XK_Begin = 0xff58 // BOL
```

Then grab it in the same loop as before:

### "Grab Keys"
```go
for i, syms := range keymap {
	backspacekeycode := xproto.Keycode(0)
	ekeycode := xproto.Keycode(0)
	// Go through all the SymsPerKeycode looking for keysym.XK_BackSpace,
	// and XK_t.
	// Grab them all.

	for _, sym := range syms {
		switch sym {
		case keysym.XK_BackSpace:
			backspacekeycode = xproto.Keycode(i)
		case keysym.XK_e:
			ekeycode = xproto.Keycode(i)
		}
	}

	if backspacekeycode != 0 {
		if err := xproto.GrabKeyChecked(
			xc,
			false,
			xroot.Root,
			xproto.ModMaskControl|xproto.ModMask1,
			backspacekeycode,
			xproto.GrabModeAsync,
			xproto.GrabModeAsync,
		).Check(); err != nil {
			log.Print(err)
		}
	}
	if ekeycode != 0 {
		if err := xproto.GrabKeyChecked(
			xc,
			false,
			xroot.Root,
			xproto.ModMask1,
			ekeycode,
			xproto.GrabModeAsync,
			xproto.GrabModeAsync,
		).Check(); err != nil {
			log.Print(err)
		}
	}
}
```

Now that we've grabbed the key, we need to add the logic to handle it:

### "Keystroke Detail Switch" +=
```go
case keysym.XK_e:
	<<<Handle E Key>>>
```

### "Handle E Key"
```go
if (key.State & xproto.ModMask1 != 0) {
	<<<Spawn A Terminal>>>
}
return nil
```

We can spawn a terminal by just calling os/exec.Command. We don't even need
to care about stdin/stdout/stderr right now.

### "Spawn A Terminal"
```go
cmd := exec.Command("xterm")
return cmd.Start()
```

### "main.go imports" +=
```go
"os/exec"
```

## Quitting Windows

We can manage windows and spawn new windows, but we can't quit them yet (unless
the application has its own quit function.) We should add a keystroke to destroy
an existing window.

The `xproto` package has a DestroyWindowChecked method, but to use it we need
to know which window to destroy. (Maybe *that's* why the other window managers
were listening for EnterNotify events..) Let's try and ensure we get notified
when a window gets focused (since we're sticking with focus-follows-pointer 
semantics, entering a window is the same as giving it focus) by adding it to
the event mask. Then, we can keep track of the active window.

### "Window Event Mask"
```go
xproto.EventMaskStructureNotify |
xproto.EventMaskEnterWindow,
```

We'll store the active window in a global.

### "window.go globals" +=
```go
var activeWindow *xproto.Window
```

and change the active window pointer whenever we get a EnterNotify event

### "X11 Event Loop Type Handlers" +=
```go
case xproto.EnterNotifyEvent:
	<<<Handle EnterNotify>>>
```

### "Handle EnterNotify"
```go
activeWindow = &e.Event
```

We should also check that we unset it when a window is destroyed.

### "DestroyEvent Handler" +=
```go
if activeWindow != nil && e.Window == *activeWindow {
	activeWindow = nil
}
```

Now, we should be able to call DestroyWindowChecked when we push some kind of
"quit" key. Let's use alt-q. (I don't think I type that very often, so it's
unlikely to conflict with other programs.)

### "Keystroke Detail Switch" +=
```go
case keysym.XK_q:
	<<<Handle q Key>>>
```

### "Handle q Key"
```go
if (key.State & xproto.ModMask1 != 0) {
	<<<Destroy Active Window>>>
}
return nil
```

### "Destroy Active Window"
```go
if activeWindow != nil {
	return xproto.DestroyWindowChecked(xc, *activeWindow).Check()
}
```

We'll have to grab the "q" key, too, before we can test this.

The GrabKeys is getting a little unwieldy and we can't just keep adding new keys
to it like this, so let's re-architecture it a little to have an array of things
we want to grab, and loop through them inside of our loop.

We need the index of every keysym that we care about, but there may be multiple
keys with the keysym, so let's keep an array of indices for each key that we care
about, loop through the keysyms looking for them, and then afterwords just grab
them all. But we also only want to grab the keys if the appropriate modifiers are
pressed, so let's include that in our structure too.


### "Grab Keys"
```go
grabs := []struct {
	sym       xproto.Keysym
	modifiers uint16
	codes     []xproto.Keycode
}{
<<<Grabbed Key List>>>
}

for i, syms := range keymap {
	for _, sym := range syms {
		for c := range grabs {
			if grabs[c].sym == sym {
				grabs[c].codes = append(grabs[c].codes, xproto.Keycode(i))
			}
		}
	}
}
for _, grabbed := range grabs {
	for _, code := range grabbed.codes {
		if err := xproto.GrabKeyChecked(
			xc,
			false,
			xroot.Root,
			grabbed.modifiers,
			code,
			xproto.GrabModeAsync,
			xproto.GrabModeAsync,
		).Check(); err != nil {
			log.Print(err)
		}

	}
}
```

### "Grabbed Key List"
```go
{
	sym:       keysym.XK_BackSpace,
	modifiers: xproto.ModMaskControl | xproto.ModMask1,
},
{
	sym:       keysym.XK_e,
	modifiers: xproto.ModMask1,
},
{
	sym:       keysym.XK_q,
	modifiers: xproto.ModMask1,
},
```

Now that we've done this refactoring, we should be able to add more keys more
easily next time.

This seems to work, except we notice that the xterms we spawned are staying
around as zombie processes, so when we spawn them, let's add a goroutine
that just waits for them to finish.

### "Spawn A Terminal"
```go
cmd := exec.Command("xterm")
err := cmd.Start()
go func() {
	cmd.Wait()
}()
return err
```

We also notice that firefox isn't actually quitting, and says it crashed when we
close with alt-q.

The (ICCCM)[https://tronche.com/gui/x/icccm/sec-4.html#s-4.2.8.1] convention that
we brushed aside earlier is a set of standardized conventions for window managers
and windows to communicate. Of particular interest (right now, at least) is the
(WM_DELETE_WINDOW)[https://tronche.com/gui/x/icccm/sec-4.html#s-4.2.8.1] section,
which is the protocol that we're supposed to follow to close windows. In particular,
it says:

> Window managers should not use DestroyWindow requests on a window that has
> WM_DELETE_WINDOW in its WM_PROTOCOLS property. 

Most of the text is written from the point of view of what a client (Window),
not the window manager, but it's enough to gather that what we should do is:

1. Check for the WM_DELETE_WINDOW atom in the WM_PROTOCOLS
2. Send a WM_DELETE_WINDOW ClientMessage event

We might want to keep the option to destroy as a last resort, but let's change
the keybinding to alt-shift-q with alt-q checking the WM_PROTOCOLS and sending
a message instead of aggressively destroying the window.

#### "Grabbed Key List" +=
```go
{
	sym:       keysym.XK_q,
	modifiers: xproto.ModMask1 | xproto.ModMaskShift,
},
```

### "Handle q Key"
```go
switch key.State {
case xproto.ModMask1:
	<<<Close window according to WM_DELETE_WINDOW protocol>>>
case xproto.ModMask1 | xproto.ModMaskShift:
	<<<Destroy Active Window>>>
}
return nil
```

Okay, so how do we check the WM_PROTOCOLS? The GetProperty(Checked) function
seems to take `xproto.Atom`s as parameters, so that might be a good place to
start

```go
func GetProperty(c *xgb.Conn, Delete bool, Window Window, Property Atom, Type Atom, LongOffset uint32, LongLength uint32)
```

We can cheat and look into what Nigel does in taowm to see how to call it:

```go
if prop, err := xp.GetProperty(xConn, false, xWin, atomWMProtocols,
	xp.GetPropertyTypeAny, 0, 64).Reply(); err != nil {
	log.Println(err)
} else if prop != nil {
	for v := prop.Value; len(v) >= 4; v = v[4:] {
		switch xp.Atom(u32(v)) {
		case atomWMDeleteWindow:
			wmDeleteWindow = true
		case atomWMTakeFocus:
			wmTakeFocus = true
		}
	}
}
```

And we'll do something similar: get the WM_PROTCOLS property,
and go through it 4 bytes (a uint32) at a time, checking for the
WM_DELETE_WINDOW atom.

Note that those consts are defined in taowm, not the xproto package.
xproto has a bunch of atom consts predefined, but not those. I suspect
it has something to do with the fact that they're part of ICCCM and not the core
definition of X11.

We'll have to do something similar early on in our initialization.

### "Initialize X"
```go
<<<Connect to X Server>>>
<<<Get Setup Information>>>
<<<Initialize Xinerama>>>
<<<Query Attached Screens>>>
<<<Set xroot to Root Window>>>
<<<Initialize Atoms>>>
<<<Take WM Ownership>>>
<<<Load KeyMapping>>>
<<<Grab Keys>>>
<<<Gather All Windows>>>
```

The xproto.InternAtom function seems to be how we get an atom that isn't
predefined.

We know we need the WM_PROTOCOLS and WM_DELETE_WINDOW atoms, so let's start
with those. We'll store them in a global variable so we can use them whenever
we need them.

### "main.go globals" +=
```go
// ICCCM related atoms
var (
	<<<Atom definitions>>>
)
```

### "Atom definitions"
```go
atomWMProtocols xproto.Atom
atomWMDeleteWindow xproto.Atom
```

### "Initialize Atoms"
```go
atomWMProtocols = getAtom("WM_PROTOCOLS")
atomWMDeleteWindow = getAtom("WM_DELETE_WINDOW")
```

### "main.go functions" +=
```go
func getAtom(name string) xproto.Atom {
	rply, err := xproto.InternAtom(xc, false, uint16(len(name)), name).Reply()
	if err != nil {
		log.Fatal(err)
	}
	if rply == nil {
		return 0
	}
	return rply.Atom
}
```

We should now have everything we need to close the window properly as discussed
above:

### "Close window according to WM_DELETE_WINDOW protocol"
```go
prop, err := xproto.GetProperty(xc, false, *activeWindow, atomWMProtocols,
	xproto.GetPropertyTypeAny, 0, 64).Reply()
if err != nil {
	return err
}
if prop == nil {
	// There were no properties, so the window doesn't follow ICCCM.
	// Just destroy it.
	<<<Destroy Active Window>>>
}
for v := prop.Value; len(v) >= 4; v = v[4:] {
	switch xproto.Atom( uint32(v[0]) | uint32(v[1]) <<8 | uint32(v[2]) <<16 | uint32(v[3]) << 24 ) {
	case atomWMDeleteWindow:
		<<<Send WM_DELETE_WINDOW message to *activeWindow>>>
	}
}
// No WM_DELETE_WINDOW protocol, so destroy.
<<<Destroy Active Window>>>
```

### "Send WM_DELETE_WINDOW message to *activeWindow">>>
```go
return xproto.SendEventChecked(
	xc,
	false,
	*activeWindow,
	xproto.EventMaskNoEvent,
	??? What is the format of this string parameter?
).Check()
```

Checking taowm for the string parameter we don't understand above, they use:

```go
string(xp.ClientMessageEvent{
	Format: 32,
	Window: xWin,
	Type:   atomWMProtocols,
	Data: xp.ClientMessageDataUnionData32New([]uint32{
		uint32(atom),
		uint32(eventTime),
		0,
		0,
		0,
	}),
}.Bytes()),
```

That's quite a handful, and we don't have the eventTime. Going back to the
[ICCCM](https://tronche.com/gui/x/icccm/sec-4.html#s-4.2.8) reference that we
used before (and is maybe where we should have started):

> 4.2.8. ClientMessage Events
> There is no way for clients to prevent themselves being sent ClientMessage events.
>
> Top-level windows with a WM_PROTOCOLS property may be sent ClientMessage
> events specific to the protocols named by the atoms in the property (see
> section 4.1.2.7). For all protocols, the ClientMessage events have the
> following:
>
>    WM_PROTOCOLS as the type field
>    Format 32
>    The atom that names their protocol in the data[0] field
>    A timestamp in their data[1] field 
>
> The remaining fields of the event, including the window field, are determined
> by the protocol.
>
> These events will be sent by using a SendEvent request with the following
> arguments:
> Argument 	Value
> destination: 	The client's window
> propagate: 	False
> event-mask: 	() empty
> event: 	As specified by the protocol

That gives us a little more context for understanding where taowm came up with
its string. In fact, now that we've read that, it's probably the only way to do
it with `github.com/BurntSushi/xgb/xproto` We don't have the eventTime field
handy, but if all that's required is "a timestamp", we can just use time.Now.

### "Send WM_DELETE_WINDOW message to *activeWindow"
```go
t := time.Now().Unix()
return xproto.SendEventChecked(
	xc,
	false,
	*activeWindow,
	xproto.EventMaskNoEvent,
	string(xproto.ClientMessageEvent{
		Format: 32,
		Window: *activeWindow,
		Type:   atomWMProtocols,
		Data: xproto.ClientMessageDataUnionData32New([]uint32{
			uint32(atomWMDeleteWindow),
			uint32(t),
			0,
			0,
			0,
		}),
	}.Bytes())).Check()
```

### "main.go imports" +=
```go
"time"
```

We can now safely quit things! (We can test this by opening a couple firefox
windows, pressing alt-q, and verifying that it prompts us with a warning that
we're closing multiple tabs, instead of a crash report.)

Next up, we'll be building on our window management by adding keys for moving
windows around in MovingWindows.md.
