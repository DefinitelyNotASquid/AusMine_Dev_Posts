# How to load 60,000 32x32 pngs in just 3 seconds... - By Mithrillia

So everyone has by now seen AusMine's Custom icons in some form or another, But do you know why we use these characters?  Why do we use the chinese and japanese languages? What the bloody hell is a unicode and why does it matter to me?

Well lets start with unicode. In the early days of computers, computer communication was just driven by the ASCII character index, however as computers started to expand globally we realised this system was not going to be accessible for everyone in the world. The ASCII Table only supports A-Z, a-z, 0-9, a few special characters like .,<>?{}() ect. Unfortunantely it wasnt really designed with expandability and accessibility at its core.

So then in the early 90's along comes Unicode, which is a universal standard that assigns a give "number" to a letter or symbol; This enables us to universally understand what someone is typing away on a keyboard, whether that be an English Qwerty keyboard, a French keyboard, a russian cyrillic keyboard or a chinese cantonese keyboard. Unicode code points can be written a few ways, you might have seen them represented in hexadecimal or alt codes before, they look like this : 0x73. Basically that the computer is doing is again representing a number or in this case a "Code Point" which when converted back to base ten (decimal) is just a number (115).

Okay so how does this relate to AusMine's custom icons and emojis?!??! Well in order to have coloured images what we do is overwrite certain sections of the unicode index in order to represent text in a particular way. Emoji's already natively live in the Unicode index, so we just assign those to the places where minecraft hasn't already done so, but for other custom ones, we assign those to "private regions" areas that have been specifically reserved so that developers can use them as they please.

In previous minecraft iterations, we were limited to the ranges of 00 - FF however these unicode pages were only able to support 16 x 16px characters and then the glyphs would only render at 16x16px (if not smaller) depending on the range's set dimensions in the glyphs.bin file.

So in order to bypass this without insane configuration and have a large unicode index set we chose the Japanese and Chinese index's to overwrite. These both were a great choice as they both attribute the folowing:

**A)** Mono spaced index's (monospace means every character is as wide and as tall as every other character)

**and**

**B)** Contain large amounts of characters as they are pictographic in nature (Meaning instead of using just a standard set of letters to make up their words, they use thousands of symbols that correspond to actual words, think sorta heiroglyphics).

This for the most part has been benificial for AusMine, however we've been limited to just 16x16px images meaning that alot of our plans to expand on this functionality had been curbed.

#### New Functionality!

In 1.16 Mojang released the ability to override greater sections of the unicode range, also we were given both the ability to resize individual unicode so, as I do i start to play around with the new formatting and play around with TTF integration. 

We also got told that the Unicode overrides using the 00 - FF has been marked as deprecated and will require Force Unicode in future versions of minecraft.

First few things that i discover, TTF (Truetype font) which was a new functionality added just doesnt support the latest and greatest standard, so it doesnt support full multicolour range, so it basically axe's any real use for this particular use case, also we can only specifiy the TTF target and not actually modify via external json any of the properties of the additional characters. Not exactly ideal...

So lets go with the other option, directly assigning a character to an image and then override it when the resource pack loads. This allows us to use the full 32-bit RGBA colour space, we can define the full properties of each Minecraft/character/Emoji/meme and generally is a nice local matching of unicode -> name and allows for debugging way better.

This can be done with some code that looks like this: 
```json
{
	"providers": [
		{
			"type": "bitmap",
			"file": "minecraft:font/1f004.png",
			"height": 7,
			"ascent": 7,
			"chars": [
				"🀄"
			]
		}
	]
}
```

### So ah, 1.16 has been out for some time Mith, why havent we seen this yet? Whats the go buddy!?!? Spill the beans

Well ah... first sorta couple of attempts initally posited some unexpected performance results...

Here is the following logs of trying to load the unicode pack with all of our icons (Approx. ~10k) using the new format that i made last year:

```
12:08:38.954 Render thread Reloading ResourceManager: UnicodePack.Zip
....
12:10:14.524 Render thread Created: 256x128x0 minecraft:textures/atlas/mob_effects.png-atlas
```

That took over 1min 30s on a 8xCore CPU, 16gb of ram, minecraft instance with the resource pack hosted locally to load. This would cripple most potatoes trying to join the server trying to use the server designated resource pack, and preferably we would like to have a seamless process of downloading and loading the resource pack for all players.

### So what went wrong?  
Well it turns out, nothing! it works exactly as specified and if we drop the icons down to about 3,000 it loads way way faster, actually only takes 6 seconds so whats slowing it down? Whats going on here, why adding an extra 7000 slows it down 15x when realistically it should only be about 10-15 seconds?

### Little bit of math theory:

In the following diagram we have some Exponential and linear curves, you dont really need to understand much about the graph but what you want to know is all those crazy ones that shoot right up to the sky after a little while are really bad performing algorithms. Given enough data, they can take 1000's of more resources or time to complete the last operation as they did the first.

Essentially, the more you throw at it, the worse it progressively gets, until at some point its not even worth running - this is the premise of why computers get slow doing the exact same task over and over again, at some point there is too much data and just not enough time.

The straight line and everything below it are examples of algorithmns that are really good at doing things no matter how much data or resources you have, you basically throw what ever at it and it either just doesnt get any worse, stays the same, or grows at a steady rate the more there is.
[![BIG O](https://res.cloudinary.com/practicaldev/image/fetch/s--NR3M1nw8--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://thepracticaldev.s3.amazonaws.com/i/z4bbf8o1ly77wmkjdgge.png "BIG O")](https://res.cloudinary.com/practicaldev/image/fetch/s--NR3M1nw8--/c_limit%2Cf_auto%2Cfl_progressive%2Cq_auto%2Cw_880/https://thepracticaldev.s3.amazonaws.com/i/z4bbf8o1ly77wmkjdgge.png "BIG O")

### Who's to blame? Me or Mojang?

Weeeeeeeeeell to be honest - its the both of us. The new format that they use is Json, its basically a nice way to define an Object and all of its properties and then specifiy particular differences between each object. its super common, no like seriously you'd be sending 500+ json requests per day just browsing and clicking through apps on your phone.

So whats wrong with it? Well json isnt exactly performant when we serialise it (Turn it from text to a Magical Code Object), and the specified operation does require the resource engine serialisation of 10,000 objects (not exactly **that** bad) and then 10,000 IO Operations (loading each image as a buffered), and then 10,000! iterations... okay i can see whats going on here.

So we need to specifiy less io and less ultimately bigger objects, but how? and is there even a way?

### Denial, Anger, Bargaining, Depression, Acceptance..... then Decompilation.

[![xx_haxor_Leet_enter_the_grid_man_xx](https://i.kym-cdn.com/entries/icons/original/000/021/807/ig9OoyenpxqdCQyABmOQBZDI0duHk2QZZmWg2Hxd4ro.jpg "xx_haxor_Leet_enter_the_grid_man_xx")](https://i.kym-cdn.com/entries/icons/original/000/021/807/ig9OoyenpxqdCQyABmOQBZDI0duHk2QZZmWg2Hxd4ro.jpg "xx_haxor_Leet_enter_the_grid_man_xx")

So if we look back at the example i showed before of the json, we can see there is actually a little more than meets the eye. If we ignore everything else and look at the "Chars" property we can see it actually takes an array - so why an array? does that mean you can just specify what characters you want to replace, and those are the ones that get replaced? So if we specified "a","b","c" any of those would be changed to stonks.png?

```json
{
	"providers": [
		{
			"type": "bitmap",
			"file": "minecraft:font/stonks.png",
			"height": 7,
			"ascent": 7,
			"chars": [
				"a","b","c"
			]
		}
	]
}
```

Well thats not actually the case - see what it would actually do is break up stonks.png into three equal widths and a, b and c would actually be represented by this. I played around with this for a while and took a peak under the hood to see what mojang was doing here, because even on the Minecraft wiki there is basically no information and i'd never actually seen any of this functionality at all before.

Turns out, if you specify a char array of an image, it will split the image into that many characters tall, basically like a sprite sheet. This meant two things:

1) we can reduce our IO Operations down quite a lot.

2) we can reduce our overall object amount down as a significant factor as well as we're going to be storing multiple Unicodes per object prop.

So after doing so perf tests, i figured out the best ratio of chars to sprite sheets seems to be at least for our testing to be 256 char's per sprite sheet. I was able to get the loading down to 3 seconds for 64k icons which means we can easily expand over the 10k we currently roll with. So we get this insanly skinny but tall png (8192px tall and 32px wide.) and our Json looks something like this:

```json
{
	"providers": [
		{
			"type": "bitmap",
			"file": "minecraft:font/sheet0.png",
			"width": 8,
			"height": 8,
			"ascent": 7,
			"chars": [
				"󰀀","󰀁","󰀂","󰀃","󰀄","󰀅","󰀆","󰀇",
				"󰀈","󰀉","󰀊","󰀋","󰀌","󰀍","󰀎","󰀏",
				"󰀐","󰀑","󰀒","󰀓","󰀔","󰀕","󰀖","󰀗",
				"󰀘","󰀙","󰀚","󰀛","󰀜","󰀝","󰀞","󰀟",
				"󰀠","󰀡","󰀢","󰀣","󰀤","󰀥","󰀦","󰀧",
				"󰀨","󰀩","󰀪","󰀫","󰀬","󰀭","󰀮","󰀯",
				"󰀰","󰀱","󰀲","󰀳","󰀴","󰀵","󰀶","󰀷",
				"󰀸","󰀹","󰀺","󰀻","󰀼","󰀽","󰀾","󰀿",
				"󰁀","󰁁","󰁂","󰁃","󰁄","󰁅","󰁆","󰁇",
				"󰁈","󰁉","󰁊","󰁋","󰁌","󰁍","󰁎","󰁏",
				"󰁐","󰁑","󰁒","󰁓","󰁔","󰁕","󰁖","󰁗",
				"󰁘","󰁙","󰁚","󰁛","󰁜","󰁝","󰁞","󰁟",
				"󰁠","󰁡","󰁢","󰁣","󰁤","󰁥","󰁦","󰁧",
				"󰁨","󰁩","󰁪","󰁫","󰁬","󰁭","󰁮","󰁯",
				"󰁰","󰁱","󰁲","󰁳","󰁴","󰁵","󰁶","󰁷",
				"󰁸","󰁹","󰁺","󰁻","󰁼","󰁽","󰁾","󰁿",
				"󰂀","󰂁","󰂂","󰂃","󰂄","󰂅","󰂆","󰂇",
				"󰂈","󰂉","󰂊","󰂋","󰂌","󰂍","󰂎","󰂏",
				"󰂐","󰂑","󰂒","󰂓","󰂔","󰂕","󰂖","󰂗",
				"󰂘","󰂙","󰂚","󰂛","󰂜","󰂝","󰂞","󰂟",
				"󰂠","󰂡","󰂢","󰂣","󰂤","󰂥","󰂦","󰂧",
				"󰂨","󰂩","󰂪","󰂫","󰂬","󰂭","󰂮","󰂯",
				"󰂰","󰂱","󰂲","󰂳","󰂴","󰂵","󰂶","󰂷",
				"󰂸","󰂹","󰂺","󰂻","󰂼","󰂽","󰂾","󰂿",
				"󰃀","󰃁","󰃂","󰃃","󰃄","󰃅","󰃆","󰃇",
				"󰃈","󰃉","󰃊","󰃋","󰃌","󰃍","󰃎","󰃏",
				"󰃐","󰃑","󰃒","󰃓","󰃔","󰃕","󰃖","󰃗",
				"󰃘","󰃙","󰃚","󰃛","󰃜","󰃝","󰃞","󰃟",
				"󰃠","󰃡","󰃢","󰃣","󰃤","󰃥","󰃦","󰃧",
				"󰃨","󰃩","󰃪","󰃫","󰃬","󰃭","󰃮","󰃯",
				"󰃰","󰃱","󰃲","󰃳","󰃴","󰃵","󰃶","󰃷",
				"󰃸","󰃹","󰃺","󰃻","󰃼","󰃽","󰃾","󰃿"
				]
		}
	]
}
```
This does mean we can only specifiy certain "sets" of characters with certain properties but this is far better of a compromise than not having Unicode characters available.

Ill be working with Waf to roll out our new Unicode api, meaning MORE emojis, MORE icons and MORE possibilities going forward, and the best part of all - its still faster than the old method we've been using!
