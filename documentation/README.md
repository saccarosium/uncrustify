# Uncrustify

## Code Overview

### Step 1: Tokenize

The entire source file is read into a buffer and then it is parsed into what I
call 'chunks'.  These are the smallest lexical elements that form keywords,
numbers, strings, identifiers, or punctuators.  The type of the chunk or token
is as descriptive as needed and may depend on the language.

The following information is gathered on each chunk:
- The token type (see [`src/token_enum.h`](../src/token_enum.h) for a complete list)
- The original line and column
- The length (character count) of the chunk
- The substring
- Whether the token was right after a tab
- The number of newlines in the string (mainly for the newline type)
- Flags (ie found in a preprocessor)

The chunk entry has other fields that get populated later.
- The parent token type
- Flags (see `cparse_types.h` for a complete list)
- Output column
- Brace Level (depth of open braces `{`)
- Level (depth of braces, parenthesis, and square brackets)

### Step 2: Tokenize Cleanup

This step changes the type of certain chunks to a simpler type is the more
complex type is not needed.  For example, the version keyword in D has two
forms:

```d
version = WIN32;
version (DEBUG)
{
   // statements
}
```

This step checks the token after the version token and changes the type of the
version chunk appropriately.  Another example is the `[` token followed by `]`.
This gets merged into a single chunk `[]`.

### Step 3: Brace Cleanup

This is probably the most complicated step in the entire program.  It figures
out the brace level/depth of each token and inserts virtual braces around
unbraced statements.   

For example, the if statement below has virtual braces inserted:

```c
if (foo) {
   bar(); }
```

This step also handles the ugliness of the `#ifdef` preprocessors in C, C++,
and C#.  To do this, the concept of a parse frame is introduced. I won't get
into details here, but the idea is that the parse frame can be pushed onto a
stack when it hits `#if` / `#else` / `#endif` preprocessors.

A big unresolved problem is what to do when you have unbalanced `#if` / `#else`
groups.  Another big item here is marking expression and statement starts and
setting the parent of parenthesis and braces.

### Step 4: Chunk Identification

Once the brace stuff is all figured out, we can do some hard-core pattern
matching to further identify each chunk.  Everything else is marked in this
step, such as:
- Variable definitions
- Function calls, prototypes, implementations
- Words are marked as types
- Operators like `*`, `-`, `+`, `--`, and `++` are classified
- Casts are identified
- The purpose of each colon (`label`, `case`, `class`, `?:`, etc) is identified

After all that is taken care of, we are ready to do useful work.

### Step 5: Brace to Virtual Brace conversion

This step converts virtual braces into real braces or, optionally, converts
real braces into virtual braces.  Obviously, this is only done on
single-statement statements.  Because of the macro abuse that C-like language
allow, this can be dangerous and break your code.

### Step 6: Inter-chunk Spacing

This step is really simple. It just goes through the list of chunks and looks
at them two at a time.  It determines whether to ignore, add or remove spaces
between the chunks.

### Step 7: Newlines 

This step inserts and/or removes newlines in key areas.
It does things like change brace styles, insert gaps between statements, etc.
Examples:

```c
if (foo) {
   bar(0);
} else {
   bar(1);
}

if (foo) 
{
   bar(0);
}
else
{
   bar(1);
}	struct pt {
   int x;
   int y;   
}

struct pt
{
   int x;
   int y;
}
```

### Step 8: Indenting

This does the obvious indenting stuff.  Indenting is done by changing the
output column of the chunk.  It is important to note that this ONLY shifts the
entire line left or right.  This step is revisited again near the end.

### Step 9: Aligning

This does all the aligning stuff.
- enums on equal signs
- structure definitions (including bit fields)
- regular assignments
- variable definitions 
- back-slash newline combos
- #define values
- trailing comments
- etc

### Step 10: Rendering

The rendering step outputs the chunks to a stream.  

This is rather simple, as the column of each chunk has already been figured
out.  The only bit of complication is that multi-line comments are formatted in
this step.
