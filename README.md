# phdr_syms
Modified version of Chris Rohlf's phdr_syms.c

Reference article: http://em386.blogspot.in/2006/10/elf-no-section-header-no-problem.html

Original article text:

ELF - No Section Header? No Problem
-----------------------------------
During the research and development process of some private anti-malware tools of mine I have come to the conclusion that there just isnt enough documentation around on ELF malware analysis. This is obviosuly because the ratio of PE to ELF malware is probably around 100 to 1. This doesnt mean unix systems don't get infected or attacked on a daily basis. It just means the attacks are either more stealthy or dont occur as often.

Infection techniques are not neccessarily what I have been researching. Its more of the obfuscation techniques that common malware uses. Just about every ELF monkey has heard of the ELF Kickers suite of tools. One of the utilities that comes with that tarball is sstrip. Its a utility for stripping the section header from an ELF binary. In Linux, the linker and loader do not require that an ELF object of type ET_EXEC has a section header, only a program header is needed. The removal of the section header is done by changing the elf header e_shnum and e_shentsize values to 0 and then stripping or zeroing out the section header itself. Unfortunately ANY tools based on the BFD libraries are hopelessly dependent on the ELF section header. And whats worse many other open source and commercial tools can be _easily_ fooled by tweaking certain values in the section header (this is sad but true, but most will blindly follow the section header details without stopping to think thats NOT what the OS loader would do!).

SStriping the section header, and any other dead code in the ELF object is an obvious no brainer for a malware author. Not only does it save size but it makes the malware that much harder to analyze. This is nothing new, sstrip has been around for 5 years. Plenty of discussion has raged over this topic. What I find most surprising is the lack of section header rebuilding tools or tools that completely ignore the section header and parse the ELF object correctly (I am told IDA does this correctly, I do not own a copy). So in this post I present to you the correct way to partially rebuild an ELF section header while grabbing some more interesting data along the way (my example is rebuilding a partial symbol table). I present this information _not_ as something new, (like I said before sstrip is 5 years old now), but as information to help others write better tools in the future.

First off lets assume the ELF object we are analyzing has had its entire section header sstriped away. Lets also assume this binary was compiled with GCC and does have some symbol relocation happening at runtime. The very first thing we want to do is parse the program header for the DYNAMIC segment. Once we have the address of the DYNAMIC segment we want to iterate over that segment and use the Elf32_Dyn struct that is defined in /usr/include/elf.h

First let me explain what the dynamic segment holds. The dynamic segment is actually the start of the .dynamic section in a gcc compiled binary. It contains a wealth of values that will help us to begin piecing back together our ELF section header. Each one of our dynamic entries is going to contain a different value stored in the d_tag member of the struct. For example in our case if dyn->d_tag == DT_REL then this means that dyn->d_un.d_val == 'start of of a reloctable section'. In the case of our gcc binary its probably the .reldyn section. Many bits and pieces of the section header can be pieced back together this way. Offsets for sections such as .interp, .ctors, .strtab, .hash, .symtab, .reldyn and .relplt can be found using the DYNAMIC segment. Most of these sections will have to be parsed to find their other section header values such as size and sh_link members etc...

I can't very well end this post without a good example. So here is how to parse the symbol tables of a binary thats been sstriped using the data from the DYNAMIC segment.

Steps:

1. Parse the program header to find the DYNAMIC segment
2. Grab the address of your strtab section by finding a DYNAMIC segment entry with d_tag type of DT_STRTAB
3. Grab the address of your symbol table by finding a DYNAMIC segment entry with d_tag type of DT_SYMTAB
4. Get the size of the string table by reading the nchains value from the Hash table which you can find from the DYNAMIC segments d_tag of type DT_HASH
5. Loop through the symbol table using the Elf32_Sym struct. Refer to the location of your string table and use the st_name value to find the string that corresponds with the symbol entry your parsing.

http://chris.rohlf.googlepages.com/phdr_syms.c.txt Please go there to find the code that accomplishes steps 1 through 5. I make no guarantee or warranty on this code whatsoever :)

One could easily build upon the code above to find the complete symbol data for a sstriped object by also parsing the relocatable sections of the binary by looking for dynamic segment entries with d_tag = DT_JMPREL || DT_RELDYN and using the Elf32_Rel structure. Thats all for now though. Thanks for reading.

UPDATE: Cleaned up some errors in the old code and now uses the hash tables nchains value to find the # of symbols
