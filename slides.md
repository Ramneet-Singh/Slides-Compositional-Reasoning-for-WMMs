---
title: "Compositional Reasoning for WMMs"
subtitle: "COV889 Course Presentation"
author:
    - Ramneet Singh \newline
    - Mrunmayi Bhalerao
institute: "IIT Delhi"
theme: "Frankfurt"
colortheme: "beaver"
#fonttheme: "professionalfonts"
#mainfont: "Hack Nerd Font"
#fontsize: 10pt
urlcolor: blue
linkstyle: bold
date: May 2023
lang: en-UK
#section-titles: false
toc: true
fontsize: 10pt
# mainfont: "gentium" # See https://fonts.google.com/ for fonts
# sansfont: "Raleway"
# monofont: "IBM Plex Mono"
mathfont: ccmath
header-includes:
 - |
  ```{=latex}
  \usepackage{amsmath,amsfonts,fancyhdr,float,array,amsthm}
  \usepackage[ligature,inference,shorthand]{semantic}
  \floatplacement{figure}{H}
  \usepackage{listings}
  \lstset{basicstyle=\ttfamily,
    showstringspaces=false,
    commentstyle=\color{red},
    keywordstyle=\color{blue}
  }
  \newcommand*{\defeq}{\stackrel{\text{def}}{=}}
  \newcommand{\beh}{\mathrm{beh}}
  %\newtheorem{theorem}{Theorem}
  %\newtheorem{lemma}{Lemma}
  \newcommand{\simplies}{\DOTSB\Longrightarrow}
  \newcommand{\Vbox}{{\setlength{\fboxsep}{0pt}\fbox{\phantom{l}}}}
  ```
---

# Introduction

## Rely/Guarantee Reasoning

- Verification of concurrent programs with shared resources is challenging due to **combinatorial explosion** 

- **Abstraction to the rescue!**

- Everything the environment can do: $\mathcal{R}$

- Everything you can do: $\mathcal{G}$

$$
\mathcal{R}, \mathcal{G} \vdash P \{\; c \;\} Q
$$ 

- *Compositional!*

## Extension to Weak Memory Models

- Judgements using earlier techniques are valid under **sequentially consistent semantics**
    - Can be directly used for data-race free code executing on weak memory models
    - But, lots of code has data races! `seqlock`, `java.util.concurrent.ConcurrentLinkedQueue` ...

- How do we extend them to weak memory models?

- What If: We could find a condition under which sequentially consistent rely-guarantee reasoning can be *soundly* preserved
     $$
     (\vdash P \{\; c \;\} Q) \,\land\, ?? \implies \vdash P \{\; c_{WM} \;\} Q
     $$ 

- Benefits:
    - Reuse existing verification techniques
    - Deal with the complexity of weak memory separately as a *side-condition*

## What are WMMs anyway?

- Relaxing the memory consistency guarantees provided by hardware enables optimisations
    - Store forwarding (will see later)
    - Write buffers

- (Part 1) **Multicopy Atomic**: One thread's stores become observable to all other threads at the same time.
    - x86-TSO, ARMv8, RISC-V

- (Part 2) **Non-Multicopy Atomic**: Each component has its own *view* of the global memory.
    - Older ARM versions, POWER, C11

- Challenge: Two types of interference now -- **Inter-Thread** + **Intra-Thread** (due to reordering)

- How will we deal with this? ...

## Teaser

- We want a compositional approach through thread-local reasoning.

- Exploit the reordering semantics of Colvin and Smith: **multicopy atomic
memory models can be captured in terms of instruction reordering.**
    - Combinatorial explosion? ($n$ reorderable instructions in a thread $\simplies$ $n!$ behaviours)
    - Introduce *reordering interference freedom* between ($\frac{n(n-1)}{2}$) pairs of instructions (Stay tuned...)

- In non-multicopy atomic WMMs, there is **no global shared state**(!!)
    - Judgement for each thread is applicable to *its view* (depends on *propagation* of writes by hardware)
    - How do we know it holds in other threads' views?
    - Represent the semantics using reordering **between** different threads
    - No longer compositional? Hardest part of the talk -- *global reordering interference freedom*: use the rely abstraction to represent reorderings between threads

# Abstract Language

## Syntax

- Individual (atomic) instructions $\alpha$

- Commands (or programs)

$$
c~:=~\epsilon~|~\alpha~|~c_1;c_2~|~c_1~\sqcap~c_2~|~c^{*}~|~c_1~||~c_2
$$ 

- Iteration, choice are non-deterministic

- Empty program $\epsilon$ represents termination

## Semantics: Commands

- Each atomic instruction $\alpha$ has a relation $\beh(\alpha)$ (over pre- and post-states) specifying its behaviour

- Program execution is defined by a small-step semantics over commands

- Iteration, non-deterministic choice are dealt with at a higher level (see next slide)

\begin{center}
\begin{tabular}{cc}
    \inference[]{}{\alpha \mapsto_{\alpha} \epsilon} & \inference[]{c_1 \mapsto_{\alpha} c_1'}{c_1;c_2 \mapsto_{\alpha} c_1';c_2} \\ \\
    \inference[]{c_1 \mapsto_{\alpha} c_1'}{c_1~||~c_2 \mapsto_{\alpha} c_1'~||~c_2} & \inference[]{c_2 \mapsto_{\alpha} c_2'}{c_1~||~c_2 \mapsto_{\alpha} c_1~||~c_2'}
\end{tabular}
\end{center}

## Semantics: Configurations

- *Configuration* $(c,\sigma)$ of a program
    - Command $c$ to be executed
    - State $\sigma$ (map from variables to values)

- *Action Step*: Performed by component, changes state
    $$
    (c,\sigma) \xrightarrow{as} (c',\sigma') \iff \exists \alpha. c \mapsto_{\alpha} c' \,\land\, (\sigma,\sigma') \in \beh(\alpha)
    $$ 

- *Silent Step*: Performed by component, doesn't change state
    $$
    (c_1 \sqcap c_2, \sigma) \rightsquigarrow (c_1,\sigma) \quad (c_1 \sqcap c_2, \sigma) \rightsquigarrow (c_2,\sigma)
    $$ 
    $$
    (c^{*},\sigma) \rightsquigarrow (\epsilon,\sigma) \quad (c^{*},\sigma) \rightsquigarrow (c;c^{*}, \sigma)
    $$ 

- *Program Step*: Action Step or Silent Step

- *Environment Step*: Performed by environment, changes state. $(c,\sigma) \xrightarrow{es} (c,\sigma')$.
