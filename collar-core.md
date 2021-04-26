---
title: 'Collar: Another Option for Lending'
author: 'Collar Development'
date: '2020-04-20 [draft version]'
---
# Introduction

In this article we introduce a decentralized protocol which establishes a lending protocol for pegged assets, providing lending and borrowing for relative-stable assets with
algorithmically set interest rates based on supply and demand.

The name 'Collar' is inspired by equity collar^[<https://en.wikipedia.org/wiki/Collar_(finance)>]:

> A collar is created by:
> 
> - buying the underlying asset
> - buying a put option at strike price, X (called the floor)
> - selling a call option at strike price, X + a (called the cap)
> 
> These latter two are a short risk reversal position. So:
> 
> Underlying - risk reversal = Collar

The core idea for Collar is:

- stable pais (no impermanent loss)
- stable interest (no uncertainty)
- stable collateral (no liquidation)

## Background

Lending protocols like Compound^[<https://compound.finance/documents/Compound.Whitepaper.pdf>] and Aave^[<https://github.com/aave/aave-protocol/blob/master/docs/Aave_Protocol_Whitepaper_v1_0.pdf>] ^[<https://github.com/aave/protocol-v2/blob/master/aave-v2-whitepaper.pdf>] provide pretty good user experience for borrowers and lenders. However, there are still challenges of current lending protocols:

- float interest

    Users are troubled with estimating capital cost of borrowing and facing liquidation risk. Current lending market depends on capital utilization deciding interest rate which can be easily manipulated by deposit whales.

- risk of frozen liquidity / ownership malfunction
    
    Suppliers of lending assets always face an embarrassing situation when the capital utilization is high enough to prevent them from withdrawing their assets. 

- irreconcilable contradiction between capital efficiency and collateral safety
  
    Borrowers always face position liquidation risk and low capital efficiency.

## Collar Features

Collar provide attractive features to both lenders and borrowers.

### For Lenders

- fixed supply interest
- low risk exposure
- liquidity providing rewards
- no liquidity frozen

### For Borrowers

- fixed borrow interest
- no liquidation risk
- 100% capital utilzation

# Collar Core Protocol

First $s_{c,a}$ is defined to denote the balance of token $c$ of account $a$ and $\mathbf{S}$ is defined to be a matrix of all $s_{c,a}$. The dimensions of $\mathbf{S}$ depends on the amount of all addresses and types of all tokens. Therefore $\mathbf{S}$ can be considered as a balance table of all users and all coins.

$$
\mathbf{S} = 
\begin{bmatrix}
s_{c_1,a_1} & s_{c_1,a_2} & \cdots & s_{c_1,a_{|\mathbb{A}|}} \\
s_{c_2,a_1} & s_{c_2,a_2} & \cdots & s_{c_2,a_{|\mathbb{A}|}} \\
\vdots & \vdots & 	\ddots & \vdots \\
s_{c_{|\mathbb{C}|},a_1} & s_{c_{|\mathbb{C}|},a_2} & \cdots & s_{c_m,a_{|\mathbb{A}|}}
\end{bmatrix}
$$

Here $\mathbb{A}$ and $\mathbb{C}$ represent the domain of all accounts and domain of all tokens. In this article an account is an blockchain wallet address^[<https://en.wikipedia.org/wiki/Ethereum#Addresses>] and a token is a fungible token^[<https://ethereum.org/en/developers/docs/standards/tokens/erc-20/>]

$\mathbf{S}$ is also marked as the followig expression in brief (attention: $\mathbb{R}_\mathtt{0}$ is required because balance should always be a non-negative number):

$$
\mathbf{S} = (s_{c,a}) \in \mathbb{R}_\mathtt{0}^{|\mathbb{C}| \times |\mathbb{A}|} 
$$

The collar core protocol can be represented as a finite state machine^[<https://en.wikipedia.org/wiki/Finite-state_machine>] and the input alphabet, state set and initial state are obvious, then only the state transition function is stated here.

Let $\mathbf{S}_t$ denotes the state $\mathbf{S}$ at time $t$ while $t \in \mathbb{N}_{\mathtt{0}}$. $f$ is used to denote collar opeation and $g$ is used to denote non collar operation. Details of $g$ will not be describled in this article because it is not releated to collar core protocol, but $g$ should always follow some basic requirments (i.e. $g$ should always satisfy ERC20^[<https://ethereum.org/en/developers/docs/standards/tokens/erc-20/>] token standard). The state transition shows below which means, the next state $\mathbf{S}_{t+1}$ can be the last state $\mathbf{S}_t$, a new state created by collar protocol or a new state follows common ERC20^[<https://ethereum.org/en/developers/docs/standards/tokens/erc-20/>] rules. :

$$
\mathbf{S}_{t+1} = \mathbf{S}_t \lor f(\mathbf{S}_t) \lor g(\mathbf{S}_t)
$$

$f$ is collar operations which can be described as:

$$
f(\mathbf{S}_t) = (f_{\mathtt{-}}(\mathbf{S}_t) \land t < t_{\mathtt{expiry}}) \lor (f_{\mathtt{+}}(\mathbf{S}_t) \land t > t_{\mathtt{expiry}})
$$

where:

$$
f_{\mathtt{-}}(\mathbf{S}_t) \in \bigcup{\begin{Bmatrix}
    {f_{\mathtt{MintDual}}{(\mathbf{S}_t, a, n)} | a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
    {f_{\mathtt{MintColl}}{(\mathbf{S}_t, a, n)} | a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
    {f_{\mathtt{BurnDual}}{(\mathbf{S}_t, a, n)} | a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
    {f_{\mathtt{BurnCall}}{(\mathbf{S}_t, a, n)} | a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
\end{Bmatrix}} \\
$$

$$
f_{\mathtt{+}}(\mathbf{S}_t) \in {f_{\mathtt{BurnColl}}{(\mathbf{S}_t, a, n)} | a \in \mathbb{A}, c \in \mathbb{C}}
$$

which means, if it is before expiry time $t_{\mathtt{expiry}}$, there are four operations (`MintDual`, `MintColl`,`BurnDual`, and `BurnCall`) can be executed by account $a$ with a parameter $n \in \mathbb{R}_{\mathtt{0}}$, otherwise operation `BurnColl` can be executed.

Before introducing the five operations (`MintDual`, `MintColl`,`BurnDual`, `BurnCall` and `BurnColl`) 3 basic operations can be defined first.

- **transfer**: $\mathbf{S}^{a_{\mathtt{s}} \overset{n}{\underset{{c}}{\longrightarrow}} a_{\mathtt{t}}}$ means account $a_{\mathtt{s}}$ transfers amount $n$ of token $c$ to account $a_{\mathtt{t}}$, which can be descibes as ($\mathbf{J}^{c, a_{\mathtt{s}}}$ is a single entry matrix^[<https://en.wikipedia.org/wiki/Single-entry_matrix>])
    
    $$
    \mathbf{S}^{a_{\mathtt{s}} \overset{n}{\underset{{c}}{\longrightarrow}} a_{\mathtt{t}}} = \mathbf{S} - n \mathbf{J}^{c, a_{\mathtt{s}}} + n \mathbf{J}^{c, a_{\mathtt{t}}}
    $$
    

- **mint**: $\mathbf{S}^{\overset{n}{\underset{{c}}{\longrightarrow}} a}$ signifies a mint operation which the account $a$ receives amount $n$ of token $c$
    
    $$
    \mathbf{S}^{\overset{n}{\underset{{c}}{\longrightarrow}} a} = \mathbf{S} + n \mathbf{J}^{c, a}
    $$

- **burn**: burn operation does a subtraction that the account $a$ burns amount $n$ of token $c$

    $$
    \mathbf{S}^{a \overset{n}{\underset{{c}}    {\longrightarrow}}} = \mathbf{S} - n \mathbf{J}^{c, a}
    $$

For simplifying and clarifying the expression $(\mathbf{S}^x)^y$ can also be marked as $\mathbf{S}^{x, y}$ and $y$ can still be a combinance of two operations which make this expression recursive

$$
\mathbf{S}^{x, y} = (\mathbf{S}^x)^y
$$

## MintDual

$f_{\mathtt{MintDual}}{(\mathbf{S}, a, n)}$ is the most important operation in collar core protocol. If account $a$ deposit amount $n$ of token $c_{\mathtt{bond}}$ into the collar pool $a_{\mathtt{pool}}$, then account $a$ will mint amount $n$ of both token $c_{\mathtt{call}}$ and token $c_{\mathtt{coll}}$.

$$
f_{\mathtt{MintDual}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a_{\mathtt{pool}},  \overset{n}{\underset{c_{\mathtt{call}}}{\longrightarrow}} a,  \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}} a}
$$

Actually the account $a$ may be a borrower. After the `MintDual` operation, the borrower can sell the received $c_{\mathtt{coll}}$ token to amount $n_{*}$ of $c_{\mathtt{want}}$. $n_{*}$ is probably slightly less than $n$ because $n - n_{*}$ is the borrowing interest. Later the borrower can burn amount $n$ of token $c_{\mathtt{call}}$ and pay amount $n$ of token $c_{\mathtt{want}}$ to the collar pool $a_{\mathtt{pool}}$ and receive its amount $n$ of token $c_{\mathtt{bond}}$ back via `BurnCall` operation before expiry time $t_{\mathtt{expiry}}$.

## MintColl

`MintColl` operation is for account $a$ to pay amount $n$ of token $c_{\mathtt{want}}$ to the collar pool $a_{\mathtt{pool}}$ and directly receives amount $n$ of token $c_{\mathtt{coll}}$. This operation can avoid shortages of token $c_{\mathtt{coll}}$ and keeps the price of token $c_{\mathtt{coll}}$ is always slightly less than token $c_{\mathtt{want}}$, otherwise arbitrageurs will swap between token $c_{\mathtt{want}}$ and token $c_{\mathtt{coll}}$.

$$
f_{\mathtt{MintColl}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{want}}}{\longrightarrow}} a_{\mathtt{pool}},  \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}} a}
$$

actually `MintColl` operation is a combination of `MintDual` operation and `BurnCall` operation which is just like a instant loan (borrow and then payback instantly).

$$
f_{\mathtt{MintColl}}{(\mathbf{S}, a, n)} = f_{\mathtt{BurnCall}}{(f_{\mathtt{MintDual}}{(\mathbf{S}, a, n)}, a, n)} 
$$

## BurnDual

A redeem operation `BurnDual` is probided to account $a$ to burn amount $n$ of both token $c_{\mathtt{call}}$ and token $c_{\mathtt{coll}}$ to get its locked token $c_{\mathtt{bond}}$ back.

Borrowers may benifits from this operation because the price of token $c_{\mathtt{coll}}$ is always slightly less than token $c_{\mathtt{want}}$ and if borrower repays much early before expiry time $t_{\mathtt{expiry}}$, it can sell token $c_{\mathtt{want}}$ to token $c_{\mathtt{coll}}$ and redeem.

$$
f_{\mathtt{BurnDual}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{call}}}{\longrightarrow}}, a \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}},a_{\mathtt{pool}} \overset{n}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a}
$$

## BurnCall

`BurnCall` is the repay operation. Account $a$ burn amount $n$ of token $c_{\mathtt{call}}$ and pay amount $n$ of token $c_{\mathtt{want}}$ to the collar pool $a_{\mathtt{pool}}$ and receive amount $n$ of token $c_{\mathtt{bond}}$.

$$
f_{\mathtt{BurnCall}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{call}}}{\longrightarrow}}, a \overset{n}{\underset{c_{\mathtt{want}}}{\longrightarrow}} a_{\mathtt{pool}},a_{\mathtt{pool}} \overset{n}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a}
$$

## BurnColl

`BurnColl` is actually for the lenders. After expiry time $t_{\mathtt{expiry}}$, token $c_{\mathtt{coll}}$ holders can burn their token $c_{\mathtt{coll}}$ can receives the underlying token $c_{\mathtt{bond}}$ and token $c_{\mathtt{want}}$ from the collar pool $a_{\mathtt{pool}}$.

$\mathbf{J}^{1, c_{\mathtt{coll}}}$ is still the single entry matrix^[<https://en.wikipedia.org/wiki/Single-entry_matrix>] and $\mathbf{J}_{|\mathbb{A}|, 1}$ is a matrix of all ones^[<https://en.wikipedia.org/wiki/Matrix_of_ones>], thus, $\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}$ represents the total supply of token $c_{\mathtt{coll}}$. Then $\frac{n}{\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}}$ is is the share of the pool.

$$
f_{\mathtt{BurnColl}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}}, a_{\mathtt{pool}} \overset{\frac{n s_{c_{\mathtt{bond}}, a_{\mathtt{pool}}}}{\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}}}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a, a_{\mathtt{pool}} \overset{\frac{n s_{c_{\mathtt{want}}, a_{\mathtt{pool}}}}{\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}}}{\underset{c_{\mathtt{want}}}{\longrightarrow}} a }
$$ 

# Analysis

## Call Token

The token $c_{\mathtt{call}}$ is the right of repayment for borrowers before expiry. It is just like buying a call option for token $c_{\mathtt{bond}}$.

Thus, after expiry the token $c_{\mathtt{call}}$ will be useless and before expiry the token $c_{\mathtt{call}}$ can be an insurance for both token $c_{\mathtt{bond}}$ and token $c_{\mathtt{want}}$ because the token $c_{\mathtt{call}}$ can always choose the higher price one between token $c_{\mathtt{bond}}$ and token $c_{\mathtt{want}}$.

## Coll Token

The token $c_{\mathtt{coll}}$ is the right of liquidation for lenders after expiry. It is just like writing a put option for token $c_{\mathtt{bond}}$.

Thus, before expiry the price of token $c_{\mathtt{coll}}$ represents the borrowing interest and after expiry the price would be around the price of token $c_{\mathtt{want}}$ if both token $c_{\mathtt{bond}}$ and token $c_{\mathtt{want}}$ pegs well.

# Conclusion

Damn a reasonable stable protocol!

# Appendix

## Notations

| symbol                                                                       | description                                   |
| ---------------------------------------------------------------------------- | --------------------------------------------- |
| $a$                                                                          | account (a.k.a address)                       |
| $a_{\mathtt{pool}}$                                                          | collar pool account                           |
| $\mathbb{A}$                                                                 | set of all accounts                           |
| $c$                                                                          | token (a.k.a coin)                            |
| $c_{\mathtt{bond}}$                                                          | bond token                                    |
| $c_{\mathtt{call}}$                                                          | call token                                    |
| $c_{\mathtt{coll}}$                                                          | coll token                                    |
| $c_{\mathtt{want}}$                                                          | want token                                    |
| $\mathbb{C}$                                                                 | set of all sorts of token                     |
| $f_{\mathtt{BurnCall}}$                                                      | `BurnCall` operation                          |
| $f_{\mathtt{BurnColl}}$                                                      | `BurnColl` operation                          |
| $f_{\mathtt{BurnDual}}$                                                      | `BurnDual` operation                          |
| $f_{\mathtt{MintColl}}$                                                      | `MintColl` operation                          |
| $f_{\mathtt{MintDual}}$                                                      | `MintDual` operation                          |
| $\mathbf{J}^{i,j}$                                                           | single entry matrix                           |
| $\mathbf{J}_{m,n}$                                                           | $(m \times n)$ dimensional matrix of all ones |
| $n$                                                                          | amount                                        |
| $\mathbb{N}_\mathtt{0}$                                                      | set of all non-negative integers              |
| $\mathbb{R}_\mathtt{0}$                                                      | set of all non-negative real number           |
| $s_{c, a}$                                                                   | balance of token $c$ of account $a$           |
| $\mathbf{S}$                                                                 | matrix of all $s_{c, a}$                      |
| $a_{\mathtt{s}} \overset{n}{\underset{{c}}{\longrightarrow}} a_{\mathtt{t}}$ | transfer operstion                            |
| $\overset{n}{\underset{{c}}{\longrightarrow}} a$                             | mint operation                                |
| $a \overset{n}{\underset{{c}}{\longrightarrow}}$                             | burn operation                                |
