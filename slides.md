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
  \newcommand{\stable}{\mathrm{stable}}
  \newcommand{\vc}{\mathrm{vc}}
  \newcommand{\sat}{\mathrm{sat}}
  \newcommand{\rif}{\mathrm{rif}}
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

# Basic Proof System

## Definitions

- Associate a verification condition $\vc(\alpha)$ with each instruction $\alpha$: Provides finer-grained control (just set to $\top$ if not needed)

- Hoare triple
$$
P \{\; \alpha \;\} Q \defeq P \subseteq \vc(\alpha) \cap \{ \sigma \,\mid\, \forall\, \sigma',\; (\sigma,\sigma') \in \beh(\alpha) \simplies \sigma' \in Q \}
$$ 

- A rely-guarantee pair $(\mathcal{R}, \mathcal{G})$ is well-formed if
    - $\mathcal{R}$ is reflexive and transitive
    - $\mathcal{G}$ is reflexive

- Stability of predicate $P$ under rely condition $\mathcal{R}$
$$
\stable_{\mathcal{R}}(P) \defeq P \subseteq \{\sigma \in P \,\mid\, \forall\, \sigma',\; (\sigma,\sigma') \in \mathcal{R} \simplies \sigma' \in P
$$ 

- Instruction $\alpha$ satisfies guarantee condition $\mathcal{G}$
$$
\sat(\alpha,\mathcal{G}) \defeq \{ \sigma \,\mid\, \forall\, \sigma',\; (\sigma,\sigma') \in \beh(\alpha) \simplies (\sigma,\sigma') \in \mathcal{G} \}
$$ 

- Now introduce rely/guarantee judgements at three levels

## Instruction Level ($\vdash_a$)

$$
\mathcal{R}, \mathcal{G} \vdash_a P \{\; c \;\} Q \defeq \stable_{\mathcal{R}}(P) \,\land\, \stable_{\mathcal{R}}(Q) \,\land\, \vc(\alpha) \subseteq \sat(\alpha,\mathcal{G}) \,\land\, P \{\; c \;\} Q
$$ 

- Interplay between environmental interference and pre-,post-conditions handled through stability

## Component Level ($\vdash_c$)

\begin{center}
    \begin{tabular}{c}
        \inference[Atom]{\mathcal{R},\mathcal{G} \vdash_a P \{\; \alpha \;\} Q}{\mathcal{R},\mathcal{G} \vdash_c P \{\; \alpha \;\} Q} 
    \\ \\   \inference[Seq]{\mathcal{R},\mathcal{G} \vdash_c P \{\; c_1 \;\} M \quad \mathcal{R},\mathcal{G} \vdash_c M \{\; c_2 \;\} Q}{\mathcal{R},\mathcal{G} \vdash_c P \{\; c_1;c_2 \;\} Q} \\ \\
        \inference[Choice]{\mathcal{R},\mathcal{G} \vdash_c P \{\; c_1 \;\} Q \quad \mathcal{R},\mathcal{G} \vdash_c P \{\; c_2 \;\} Q}{\mathcal{R}, \mathcal{G} \vdash_c P \{\; c_1~\sqcap~c_2 \;\} Q}
    \\ \\   \inference[Iteration]{\mathcal{R},\mathcal{G} \vdash_c P \{\; c \;\} P \quad \stable_{\mathcal{R}}(P)}{\mathcal{R},\mathcal{G} \vdash_c P \{\; c^{*} \;\} Q} \\ \\
        \inference[Conseq]{\mathcal{R},\mathcal{G} \vdash_c P \{\; c \;\} Q \quad P' \subseteq P \quad \mathcal{R'} \subseteq \mathcal{R} \quad Q \subseteq Q' \quad \mathcal{G} \subseteq \mathcal{G'}}{\mathcal{R'}, \mathcal{G'} \vdash_c P' \{\; c \;\} Q'}
    \end{tabular}
\end{center}

## Global Level ($\vdash$)

- Global satisfiability needs component satisfiability + **interference check**
$$
\inference[Comp]{\mathcal{R}, \mathcal{G} \vdash_c P \{\; c \;\} Q \quad \rif(\mathcal{R},\mathcal{G},c)}{\mathcal{R},\mathcal{G} \vdash P \{\; c \;\} Q}
$$ 

- Usual parallel rule
$$
\inference[Par]{\mathcal{R}, \mathcal{G} \vdash_c P \{\; c \;\} Q \quad \rif(\mathcal{R},\mathcal{G},c)}{\mathcal{R},\mathcal{G} \vdash P \{\; c \;\} Q}
$$ 

# Multicopy Atomic Memory Models

## Reordering Semantics: Basics

- Multicopy atomic memory models can be characterised using a *reordering* relation $\hookleftarrow$ over pairs of instructions in a component

- $\hookleftarrow$ is syntactically derivable based on the specific memory model. E.g., in ARMv8
    - Two instructions which don't access (read or write) a common variable can be reordered
    - Various types of memory barriers prevent reordering

- *Forwarding* is another complication
    - $\beta = \texttt{x := 3} ; \alpha = \texttt{y := x}$. Can forward the value $3$ to $y$, losing dependence between $\alpha,\beta$.
    - `x := 3 ; y := x` $\Longrightarrow$ `y := 3 ; x := 3`
    - Denote $\alpha$ with the value written in an earlier instruction forwarded to it as $\alpha_{<\beta>}$.

- Forwarding may continue arbitrarily and can span multiple instructions

## Reordering Semantics: Formal

- $\alpha_{<c>}$: cumulative forwarding effects of the instructions in command $c$ on $\alpha$

- Ternary relation $\gamma < c < \alpha$: Reordering of instruction $\alpha$ prior to command $c$, with cumulative forwarding effects producing $\gamma$.

- Definition by induction
\begin{align*}
\alpha_{<\beta>} < \beta < \alpha &\defeq \beta \hookleftarrow \alpha_{<\beta>} \\
\alpha_{<c_1;c_2>} < c_1;c_2 < \alpha &\defeq \alpha_{<c_1;c_2>} < c_1 < \alpha_{<c_2>} \,\land\, \alpha_{<c_2>} < c_2 < \alpha
\end{align*}

- Example: $\alpha = (\texttt{y := x}), \beta = (\texttt{x := 3}), \gamma = (\texttt{z := 5})$. $\alpha_{<\beta>} = (\texttt{y := 3})$,$\alpha_{<\gamma ; \beta>} = (\texttt{y := 3})$.
    $$
    \texttt{y := 3} < \texttt{x := 3} < \texttt{y := x} \quad \texttt{y := 3} < \texttt{z := 5 ; x := 3} < \texttt{y := x}
    $$

- Can execute an instruction which occurs later in the program if reordering and forwarding can bring it (in its new form $\gamma$) to the beginning
$$
\inference[Reorder]{c_2 \mapsto_{\alpha} \quad \gamma < c < \alpha}{c_1 ; c_2 \mapsto_{\gamma} c_1 ; c_2'}
$$ 

## Reordering Interference Freedom


