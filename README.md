# Tips for using AI to help with MAME development
Hints and tips for using AI assistance with MAME.  Version 3.6, August 19, 2026.

**WARNING**: so-called "vibe coding" is *not acceptable* for MAME.  You're welcome to use it for personal things for yourself, but for doing actual submittable MAME work you need some experience with programming and the ability to understand and edit what the AI models output.

Also, please write the submission comment yourself.  You can restate things the AI model said, but the AI model spew is 10 times more effort for whoever is reviewing your submission.  And follow MAME's [official AI guidelines](https://docs.mamedev.org/contributing/index.html).  (They're at the bottom of that page).

One stylistic note: when typing prompts, I bracket file and pathnames with backticks so that names with spaces in them aren't ambiguous.  Here in Markdown land that translates to the `code style` with a gray background.  I found that appropriate so I've kept it.

## What models can I use?
Any of the current frontier or near-frontier models have given good results.  I lack the local hardware to effectively run any of the high-end open weights models so my suggestions will stick to the well-known closed ones: **Sonnet 5**, **Opus 5**, or **Fable 5** from [Anthropic](https://claude.ai/), **GPT-5.6** from [OpenAI](https://openai.com/), or **Grok 4.6** from [SpaceX AI](https://x.ai/).  I have personally done useful MAME work with each of the listed models.

Note that the version listed is important!  **Opus 4.8** can and has done useful MAME work but is much more likely to go haywire.  **GPT 4.5** is when GPT started getting really good for code, and 4.6 is quite a bit better in my testing.  Similarly, **Grok 4.5** is the first version that's able to do good quality MAME work, and 4.6 is a significant upgrade over that.

I plan to evaluate **Kimi K3** from [Moonshot AI](https://moonshot.ai/) as soon as I make it through their waiting list.  As advertised it has similar tradeoffs to **Grok 4.5**: not at the latest frontier capability, but substantially less expensive per token.

I am currently evaluating **Muse Spark** from [Meta](https://developer.meta.com/ai/products/muse-code/).  My early verdict: it's not as good as the latest and greatest models but it's definitely capable, and very cheap.  You will definitely need to hold it's hand a lot more, but it's a good and relatively inexpensive way to dip your toes into AI assisted development.  I appreciate the pay-as-you-go setup: you can see a live accounting of how much you owe and it doesn't actually bill you until it hits US$20.

## How do I get started?
Each model vendor has a program called a "harness" which enables their models to work with programs and data on your local machine.  Anthropic's is **Claude Code**, OpenAI's is **Codex** (although Codex is now being merged with their general ChatGPT app and I'm unclear what the final branding is going to be), and SpaceX AI's is **Grok Build**.  Installation instructions are available on each vendor's site, but typically there's a command line to copy/paste for macOS or Linux and an installer for Windows.

You will also need a paid account to do any kind of real work.  The vendors have settled in mostly at around US$20 per month as the going rate for a decent amount of usage on the coding models and harnesses.  Anthropic's Pro account at US$17 gets you decent limits for MAME usage.  OpenAI's US$8/month Go account gets your foot in the door with Codex, and the US$20/month Plus account gives you GPT-5.6 access and better limits.  SpaceX AI's Free plan lets you actually use Grok Build on a limited basis for free, and their next tier up is the US$30/month SuperGrok account.  There's also an unofficial tier between those two where you get some usage if you have a Premium X/Twitter account.

## What can I do with AI and MAME?
Here are some things I've used AI models for with MAME development.  Prompts are similar to what I really used to do these things, but not necessarily exact.  In some cases the prompts reflect knowledge I didn't have when I did that actual thing.

### Bug tracing

Got funky bugs?  Describe the bug to the model and let it trace the issue.

**Sample prompt:** *When I boot the Power Macintosh 7200 in MAME with Mac OS 7.6.1 performance gets very bad.  My command line is `mame pmac7200 mac761`.  You can use MAME's -rtc option to make runs deterministic, and MAME's Lua boot scripting to automate MAME*

### Static firmware analysis

Need a starting point to create or refine a skeleton driver?  Let the model do the initial exploration.

**Sample prompt:** *`~/s3000xl.bin` is the firmware image for an Akai S3000XL rackmount MIDI sampler, which uses the NEC V53 (x86) CPU.  MAME unidasm is available at `~/mame/unidasm` for disassembly, and remember that numeric arguments to it are assumed to be decimal unless you prefix with 0x.  Perform an analysis of the operation of the firmware and create a report with the memory and I/O port maps.*

### Correctness verification

Know that there are bugs in a MAME device or component but not sure where to look?  The model can reduce the tedious searching.

**Sample prompt:** *Please verify the operation of MAME's M680x0 FPU emulation in `src/devices/cpu/m68000`.  Use the Motorola manuals for reference.  The 68881 manual is at `~/Documents/m68k/68881_Users_Guide.pdf`, the 68030 manual is at `~/Documents/m68k/MC68030_Users_Manual.pdf`, and the 68040 manual is at `~/Documents/m68k/MC68040_Users_Manual.pdf`.  Create a report of possible issues found.*

### Directional guidance

You know what you want to do, but you're not sure how to do it?  The model can give some ideas.

**Sample prompt:** *I want to emulate the Apple MESH SCSI controller.  It's a pretty heavily customized version of the common NCR5394, and a programming manual is at `~/Documents/Mac/MESH_Users_Guide.pdf`.  Would it be reasonable to subclass the 5394 in `src/devices/machine/ncr53c90.cpp` for this, or would it be better to create a separate nscsi device?*

### Mechanical refactoring

You've got working code, but it's in the wrong form to move forward.  Or you realize you've made a grave architectural mistake but now there's a few thousand lines of working code.  The model is happy to do it for you and you can do something less likely to aggravate your repetitive strain injury.

**Sample prompt:** *For asc_easc_device, move the FIFO popping and status update logic from sound_stream_update() to a new function called pop_fifo().*

### Understanding a subsystem

You don't quite understand how something works in MAME and you want to get your head around it before attempting to change it.

**Sample prompt:** *Write an explanation of how inputs work in MAME.*

## Things I've learned
- You get better results on multi-step tasks by asking the model to create a plan for what you want first.  That also gives you an opportunity to review the plan and issue corrections and clarifications.
- When sending the model on a bug hunt, not every change it makes will turn out to be important once you get to success.  It's always a good idea to try reverting each of the changes afterwards to find out what was important.  The Mac II A/UX patch I was sent originally was much, much more invasive than what actually landed.  (Admittedly, the one to macscsi.cpp was more involved but it also fixed A/UX on all 5380 machines).
- Models can and do get off on a tangent that's not useful towards solving your problem, especially when performing bug tracing.  Don't be afraid to hit Esc to stop them and offer a correction.  Codex has the "nudge" feature for exactly this reason.
- The longer a session runs and the more context builds up, the dumber the model gets.  Don't be afraid to ask it to create a handoff document, and then use that handoff document to seed a new, clean session that will "think more clearly".
- When you are having the model generate code, tell it in advance what your preferred bracing style is (GNU, or Allman, or "follow the style of the rest of the file").
- Don't be afraid to ruthlessly edit the comments it adds.  Models output a paragraph where a sentence will do and a sentence in cases where even a junior programmer can see what's happening.  Including something like "Keep comments limited to tricky or unclear algorithms, and don't justify the changes" in your prompt can help too.
- If you're in a work tree that you don't want to submit directly from, tell it to not do any git write operations.

## When submitting to MAME
- Take a final pass over the code.  Make sure comments are useful and not just spewing.  Anthropic's models in particular love to basically apologize on bended knee for every line changed, and that's unnecessary and off-putting.
- Write the pull request description yourself.  At the very least, take a machete to whatever your model generated because it's probably excessive.  A bunch of show-and-tell in the description is not necessary (if you include a table, you're probably doing it wrong).  Just tell us what changed and what it fixed.  MAME's historical readme.txt files provide a good idea of what good taste looks like in pull request descriptions.
- Don't forget your AI usage disclosure.
