# `make`/`Makefile` magic


## Question: [How to check if a file from a list contains something, in makefile?](https://stackoverflow.com/q/45055647/404556)

I want to check if all the files that have a specific name have in them a string, if not to report
them. I wrote this sequence and tried multiple some others, but I don't know how to access the
contents of a file from list.

    SOURCES := $(shell find $(SOURCEDIR) -name 'mod.mak')
    $(foreach File, Files,
        $(if $(grep -q "aaabbb" "$File"),,@echo "WARNING Missing sequence")
    )


## Answer

You have multiple issues with your script. 

First of all, you [need some rule/target](https://www.gnu.org/software/make/manual/make.html#Makefile-Contents). For your example we can make a [`PHONY` target](https://www.gnu.org/software/make/manual/make.html#Phony-Targets) `test`. Second, to iterate over values in `SOURCES`, you need to [reference it](https://www.gnu.org/software/make/manual/make.html#Reference) as `$(SOURCES)`. Similarly for `$(file)` in call to `grep`. Also, `make`'s [`if`](https://www.gnu.org/software/make/manual/make.html#Conditional-Functions) is interpreting value, not exit code, so you shouldn't silence `grep`.

This will do it:

    .PHONY: test
    
    SOURCES := $(shell find "$(SOURCEDIR)" -name 'mod.mak')
    
    test:
        $(foreach file,$(SOURCES),$(if $(shell grep "aaabbb" "$(file)"),,@echo "WARNING Missing sequence in $(file)"))

