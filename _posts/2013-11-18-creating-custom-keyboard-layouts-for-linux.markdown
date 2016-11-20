---
layout: post
title: "Creating custom keyboard layouts for Linux"
date: 2013-11-18 21:05
comments: true
published: true
categories: [linux, keyboard, unicode]
---

I think almost every one of you has some beef with your keyboard layout. There are characters you're never going to use, there are characters you'd love to use but they're missing (and you don't like/can't use the compose key for them), and so on.

For example, this is the default Polish layout in Linux:

![My helpful screenshot]({{ root_url }}/images/layout-pl.png)

A keen eye can notice several things that could be improved:

* many characters can be input in multiple ways: `[` (as `[` or as `AltGr-9`), `ł` (as `AltGr-w` or `AltGr-l`), `&` (as `Shift-7` or `AltGr-Shift-k`), `@` (as `Shift-2` or `AltGr-q`),

* the layout has some unassigned combinations (for example `AltGr-Shift-4`),

* some important characters are missing: `€` (the euro sign) and `„` (Polish opening quote),

* since I can use the compose key, I don't need all those arbitrarily assigned dead keys on the right.

Of course your keyboard layout may have different problems.

**Disclaimer:** The following was done on Ubuntu 12.04 with Unity DE, I don't guarantee it will work on other distros. In particular, the file paths can differ.

How to create a new one, you ask? That's actually pretty simple.

* layout definitions are in the `/usr/share/X11/xkb/symbols` directory,

* layout metadata are in the `/usr/share/X11/xkb/rules/evdev.xml` file.

Let's first define our layout. The `/usr/share/X11/xkb/symbols` contains files, each corresponding to a language. Some layouts can belong to several languages, but they're defined in only one of the files. If your layout is supposed to be listed under one language in the keyboard setting anyway, you'll have to add it to the correct file. Since my keyboard layout was to be mostly an improvement on top of the Polish language layout, I decided to add it to `/usr/share/X11/xkb/symbols/pl`.

<!-- more -->

The format of this file is pretty simple. I'm going to analyse the default Polish layout line by line.

```
partial default alphanumeric_keys
xkb_symbols "basic" {
```

`default` means that this is the default layout in this file (other layouts don't have this tag).

`"basic"` is the indentifier of this layout within the file.

```
    include "latin"
```

Since most layouts are similar, the `include` directive simply inserts all key definitions from one layout to another. This one includes the (default) `"basic"` layout from the `/usr/share/X11/xkb/symbols/latin` file.

```
    name[Group1]="Polish";
```

No clue what it does. It's recommended to set it to a full English name for the layout.

```
    key <AD01>  { [         q,          Q ] };
    key <AD02>  { [         w,          W ] };
    key <AD03>  { [         e,          E,      eogonek,      Eogonek ] };
    key <AD09>  { [         o,          O,       oacute,       Oacute ] };

    key <AC01>  { [         a,          A,      aogonek,      Aogonek ] };
    key <AC02>  { [         s,          S,       sacute,       Sacute ] };
    key <AC04>  { [         f,          F ] };

    key <AB01>  { [         z,          Z,    zabovedot,    Zabovedot ] };
    key <AB02>  { [         x,          X,       zacute,       Zacute ] };
    key <AB03>  { [         c,          C,       cacute,       Cacute ] };
    key <AB06>  { [         n,          N,       nacute,       Nacute ] };
```

Those are the definitions that make the Polish layout different from the default Latin one.Each line contains: 

* the `key` keyword,

* a symbol defining the physical key,

* a list of characters associated with that key, from left to right: the default one, with Shift key, with AltGr key, with both Shift and AltGr keys.

The names of the physical keys are as follows (I'm referring to keys by their American/Polish usage):

* `<TLDE>` for the grave accent/tilde key

* `<BKSL>` for the backslash/bar key

* `<AB01>`, `<AB02>` and so on for `Z`, `X` ... keys

* `<AC01>`, `<AC02>` and so on for `A`, `S` ... keys

* `<AD01>`, `<AD02>` and so on for `Q`, `W` ... keys

* `<AE01>`, `<AE02>` and so on for `1`, `2` ... keys

Note: this refers to their *physical* layout, so I could also say that `AB` is the row above the spacebar, `AC` is the row with caps lock key, `AD` is the row with tab key, and `AE` is the row with digits, and the numbering goes from left to right. For example, `<AD11>` is labelled `[` on the American keyboard and `Ü` on the German one; `<AB01>` is `Z` on the American keyboard, `W` on the French keyboard, and `Y` on the German keyboard. `<TLDE>` is always in the top left, just under the `Esc` key, and `<BKSL>` is in either `AC`, `AD` or `AE` row (depending on the keyboard).

Also, for some reason trying to override a key with a shorter definition doesn't work.

As for the character codes, you can look around the files to learn them, or just use Unicode codepoints in hexadecimal, for example `U0041` and `A` are synonymous.

```
    include "kpdl(comma)"
```

This defines the behaviour of the decimal point key on the numpad. (More accurately, imports such behaviour override from `/usr/share/X11/xkb/symbols/kpdl`.)

```
    include "level3(ralt_switch)"
};
```

And finally, this defines how the 3rd level (the one with accented characters, the one I referred to as “AltGr”) is accessed. In this case, it's with the right Alt key.

So I created my own layout by copying the existing one and modifying it:

```
partial alphanumeric_keys
xkb_symbols "custom3" {
    include "latin"
```

`"custom3"` is the identifier of my new layout (why 3, I'll explain later). My new layout is not defined as the default one.

```
    name[Group1]="Polish (custom)";
```

No clue why I did this, but it looks consistent with the rest of the file, so I guess it's correct.

```
    key <AE07>  { [ 7, ampersand, copyright, seveneighths ] };
    key <AE08>  { [ 8, asterisk, U221E, trademark ] };
    key <AE09>  { [ 9, parenleft, guillemotleft, plusminus ] };
    key <AE10>  { [ 0, parenright, guillemotright, degree ] };

    key <AD03>  { [         e,          E,      eogonek,      Eogonek ] };
    key <AD09>  { [         o,          O,       oacute,       Oacute ] };

    key <AC01>  { [         a,          A,      aogonek,      Aogonek ] };
    key <AC02>  { [         s,          S,       sacute,       Sacute ] };
    key <AC04>  { [         f,          F ] };

    key <AB01>  { [         z,          Z,    zabovedot,    Zabovedot ] };
    key <AB02>  { [         x,          X,       zacute,       Zacute ] };
    key <AB03>  { [         c,          C,       cacute,       Cacute ] };
    key <AB06>  { [         n,          N,       nacute,       Nacute ] };

    include "kpdl(comma)"

    include "level3(ralt_switch)"


    key <TLDE>  { [ grave, asciitilde, notsign, section ] };

    key <AE04>  { [ 4, dollar, onequarter, EuroSign ] };
    key <AE07>  { [ 7, ampersand, copyright, seveneighths ] };
    key <AE08>  { [ 8, asterisk, U221E, trademark ] };
    key <AE09>  { [ 9, parenleft, guillemotleft, plusminus ] };
    key <AE10>  { [ 0, parenright, guillemotright, degree ] };
    key <AE11>  { [ minus, underscore, U2013, questiondown ] };
    key <AE12>  { [ equal, plus, U2260, Greek_SIGMA ] };

    key <AD01>  { [ q, Q, Greek_pi, Greek_OMEGA ] };
    key <AD02>  { [ w, W, cent, U20A9 ] };
    key <AD11>  { [bracketleft,  braceleft, udiaeresis, Udiaeresis ] };
    key <AD12>  { [bracketright, braceright, ssharp,  ediaeresis ] };

    key <BKSL>  { [backslash, bar, eacute, egrave] };

    key <AC07>  { [ j, J, doublelowquotemark, singlelowquotemark] };
    key <AC08>  { [ k, K, kra, U2030] };
    key <AC10>  { [ semicolon,    colon, odiaeresis, Odiaeresis ] };
    key <AC11>  { [apostrophe, quotedbl, adiaeresis,  Adiaeresis ] };

    key <AB08>  { [ comma,  less, ellipsis, multiply ]  };
    key <AB10>  { [ slash, question, agrave, idiaeresis ] };

};
```

As you can see, I added an euro sign to `Shift-AltGr-4`, opening quotes to the `J` key, and so on.

Okay, my layout is ready. Time to hook it in.

Now we're editing the `/usr/share/X11/xkb/rules/evdev.xml` file. The interesting fragment looks like this:

```xml
    <layout>
      <configItem>
        <name>pl</name>
        <shortDescription>pl</shortDescription>
        <description>Polish</description>
        <languageList>
          <iso639Id>pol</iso639Id>
        </languageList>
      </configItem>
      <variantList>
        <variant>
          <configItem>
            <name>qwertz</name>
            <description>Polish (qwertz)</description>
          </configItem>
        </variant>
```

What does it mean? `configItem/name` is the name of the file in `/usr/share/X11/xkb/symbols` that contains layout definitions. `languageList` is a list of languages which that file supports. `variantList` is a list of alternative layouts that are defined in that file and their user-friendly names (which will be translated if there's translation for them).

Since I didn't want to add the translation for the name of my layout, I simply added an entry with the name hardcoded in Polish:

```xml
        <variant>
          <configItem>
            <name>custom3</name>
            <description>Polski (własny)</description>
          </configItem>
        </variant>
```

And that's all!

Also, it's time to explain why it's `custom3`, not `custom`. Since those files are cached in memory by the DE, if you change the layout in the `/usr/share/X11/xkb/symbols/*` file, those changes won't be reloaded. Since I didn't want to restart everything just to test the layout, I solved that problem by adding 2 (and later 3) to the identifier.

And this is the result:

![My helpful screenshot]({{ root_url }}/images/layout-pl-custom.png)

Still not perfect, but way better.