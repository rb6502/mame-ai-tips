# Tips for using AI to help with MAME development
Hints and tips for using AI assistance with MAME.  Version 2, July 29, 2026.

**WARNING**: so-called "vibe coding" is *not acceptable* for MAME.  You're welcome to use it for personal things for yourself, but for doing actual submittable MAME work you need some experience with programming and the ability to understand and edit what the AI models output.

Also, please write the submission comment yourself.  You can restate things the AI model said, but the AI model spew is 10 times more effort for whoever is reviewing your submission.  And follow MAME's [official AI guidelines](https://docs.mamedev.org/contributing/index.html).  (They're at the bottom of that page).

One stylistic note: when typing prompts, I bracket file and pathnames with backticks so that names with spaces in them aren't ambiguous.  Here in Markdown land that translates to the `code style` with a gray background.  I found that appropriate so I've kept it.

## What model should I use?
Any of the current frontier or near-frontier models have given good results.  I lack the local hardware to effectively run any of the high-end open weights models so my suggestions will stick to the well-known ones: **Sonnet 5**, **Opus 5**, or **Fable 5** from [Anthropic](https://claude.ai/), **GPT-5.5** or **GPT-5.6** from [OpenAI](https://openai.com/), or **Grok 4.5** from [SpaceX AI](https://x.ai/).  I have personally done useful MAME work with each of the listed models.

Note that the version listed is important!  **Opus 4.8** can and has done useful MAME work but is much more likely to go haywire.  **GPT 4.5** is when GPT started getting really good for code, and 4.6 is of course better.  Similarly, **Grok 4.5** is the first version that's able to do good quality MAME work.  It's not as good as **Opus 5** or **GPT 4.6** but it's much cheaper per token and it's a worthwhile tradeoff in my experience.

I plan to evaluate **Kimi K3** from [Moonshot AI](https://moonshot.ai/) soon.  As advertised it has similar tradeoffs to **Grok 4.5**: not at the latest frontier capability, but substantially less expensive per token.

## How do I get started?
Each model vendor has a program called a "harness" which enables their models to work with programs and data on your local machine.  Anthropic's is **Claude Code**, OpenAI's is **Codex** (although Codex is now being merged with their general ChatGPT app and I'm unclear what the final branding is going to be), and xAI's is **Grok Build**.  Installation instructions are available on each vendor's site, but typically there's a command line to copy/paste for macOS or Linux and an installer for Windows.

You will also need a paid account to do any kind of real work.  If you have a Premium or better X/Twitter account you automatically get some Grok usage.  The vendors have settled in mostly at around US$20 per month as the going rate for a decent amount of usage on the coding models and harnesses.  Anthropic's Pro account at US$17 gets you decent limits for MAME usage.  OpenAI's US$8/month Go account gets your foot in the door with Codex, and the US$20/month Plus account gives you GPT-5.6 access and better limits.  SpaceX AI's Free plan lets you actually use Grok Build on a limited basis for free, and their next tier up is the US$30/month SuperGrok account.

## What can I do with AI and MAME?
Here are some things I've used AI models for with MAME development.  Prompts are similar to but not necessarily exact, and in some cases reflect knowledge I didn't have when I did that actual thing.

### Bug tracing

Got funky bugs?  Describe the bug to the model and let it trace the issue.

**Sample prompt:** *When I boot the Power Macintosh 7200 in MAME with Mac OS 7.6.1 performance gets very bad.  My command line is `mame pmac7200 mac761`.  You can use MAME's -rtc option to make runs deterministic, and MAME's Lua boot scripting to automate MAME*

### Static firmware analysis

Need a starting point to create or refine a skeleton driver?  Let the model do the initial exploration.

**Sample prompt:** *~/s3000xl.bin is the firmware image for an Akai S3000XL rackmount MIDI sampler, which uses the NEC V53 (x86) CPU.  MAME unidasm is available at `~/mame/unidasm` for disassembly, and remember that numeric arguments to it are assumed to be decimal unless you prefix with 0x.  Perform an analysis of the operation of the firmware and create a report with the memory and I/O port maps.*

### Correctness verification

Know that there are bugs in a MAME device or component but not sure where to look?  The model can reduce the tedious searching.

**Sample prompt:** *Please verify the operation of MAME's M680x0 FPU emulation in `src/devices/cpu/m68000`.  Use the Motorola manuals for reference.  The 68881 manual is at `~/Documents/m68k/68881_Users_Guide.pdf`, the 68030 manual is at `~/Documents/m68k/MC68030_Users_Manual.pdf`, and the 68040 manual is at `~/Documents/m68k/MC68040_Users_Manual.pdf`.  Create a report of possible issues found.*

### Directional guidance

You know what you want to do, but you're not sure how to do it?  The model can give some ideas.

**Sample prompt:** *I want to emulate the Apple MESH SCSI controller.  It's a pretty heavily customized version of the common NCR5394, and a programming manual is at `~/Documents/Mac/MESH_Users_Guide.pdf`.  Would it be reasonable to subclass the 5394 in `src/devices/machine/ncr53c90.cpp` for this, or would it be better to create a separate nscsi device?*

### Mechanical refactoring

You've got working code, but it's in the wrong form to move forward.  Or you realize you've made a grave architectural mistake but now there's a few thousand lines of working code.  The model is happy to do it for you and you can do something less likely to aggravate your repetitive strain injury.

**Sample prompt:** *For asc_easc_device, move the FIFO popping and status update logic from sound_stream_update() to a new function called pop_fifo().*

## Things I've learned
- Models can and do get off on a tangent that's not useful towards solving your problem, especially when performing bug tracing.  Don't be afraid to hit Esc to stop them and offer a correction.  Codex has the "nudge" feature for exactly this reason.
- The longer a session runs and the more context builds up, the dumber the model gets.  Don't be afraid to ask it to create a handoff document, and then use that handoff document to seed a new, clean session.
- When you are having the model generate code, tell it in advance what your preferred bracing style is (GNU, or Allman, or "follow the style of the rest of the file").  And don't be afraid to ruthlessly edit the comments it adds.  Models output a paragraph where a sentence will do and a sentence in cases where even a junior programmer can see what's happening.
