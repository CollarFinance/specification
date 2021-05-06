---
title: 'Collar: Optimized Stablecoin Lending'
author: 'Collar Development Team'
date: 'Version 0.1 May 2021'
---
# Introduction

Decentralized money markets centered on asset-backed cryptocurrencies form the foundation of yield farming and are important components of the DeFi ecosystem. Pool-based lending protocols like Compound^[<https://compound.finance/documents/Compound.Whitepaper.pdf>] and Aave^[<https://github.com/aave/aave-protocol/blob/master/docs/Aave_Protocol_Whitepaper_v1_0.pdf>] ^[<https://github.com/aave/protocol-v2/blob/master/aave-v2-whitepaper.pdf>] provide functional, convenient systems for borrowers and lenders. However, users of these and other current lending protocols are still presented with significant challenges:

- floating interest

    Borrowers are burdened with estimating the capital cost of borrowing and the risk of liquidation. Current lending markets depend on capital utilization to decide interest rates; this mechanism is vulnerable to manipulation by deposit whales.

- risk of frozen assets
    
    Lenders have no protection from the scenario where capital utilization is high enough to prevent them from withdrawing their assets. 

- dilemma between capital efficiency and collateral safety
  
    Borrowers must accept greater liquidation risk to achieve greater capital efficiency.

These challenges are present due to the general (asset-agnostic) nature of the protocols. In this paper we introduce a decentralized lending protocol, Collar, which establishes money markets optimized for pegged assets, providing lending and borrowing functions with fixed interest rates based on supply and demand.

The name Collar is inspired by the *collar* option strategy^[<https://en.wikipedia.org/wiki/Collar_(finance)>]: This flexible hedging option is created by holding the underlying, buying an out of the money put option, and selling an out of the money call option. 
This strategy limits the range of possible positive or negative returns on the underlying to a specified range. 

The core design goals of Collar are:

- stable pairs (no impermanent loss)
- stable interest (no uncertainty)
- stable collateral (no liquidation)

Collar is able to meet these goals by optimizing the lending and borrowing mechanism for pegged assets, and thus is able to overcome the previously discussed challenges.

## Collar Features

The Collar protocol establishes money markets that provide attractive features to both lenders and borrowers.

### For Lenders

- fixed supply interest
- low risk exposure
- liquidity providing rewards
- no frozen liquidity

### For Borrowers

- fixed borrow interest
- no liquidation risk
- 100% capital utilization

# Collar Core Protocol

First, $s_{c,a}$ is defined to denote the balance of token $c$ of account $a$ and $\mathbf{S}$ is defined to be a matrix of all $s_{c,a}$. The dimensions of $\mathbf{S}$ depend on the total numbers of accounts and types of tokens. The matrix $\mathbf{S}$ can be considered a balance table of all users and all tokens.

$$
\mathbf{S} = 
\begin{bmatrix}
s_{c_1,a_1} & s_{c_1,a_2} & \cdots & s_{c_1,a_{|\mathbb{A}|}} \\
s_{c_2,a_1} & s_{c_2,a_2} & \cdots & s_{c_2,a_{|\mathbb{A}|}} \\
\vdots & \vdots & 	\ddots & \vdots \\
s_{c_{|\mathbb{C}|},a_1} & s_{c_{|\mathbb{C}|},a_2} & \cdots & s_{c_m,a_{|\mathbb{A}|}}
\end{bmatrix}
$$

Here, $\mathbb{A}$ and $\mathbb{C}$ represent the domains of all accounts and all types of tokens, respectively. In this paper, an *account* is a blockchain wallet address^[<https://en.wikipedia.org/wiki/Ethereum#Addresses>] and a *token* is a fungible token^[<https://ethereum.org/en/developers/docs/standards/tokens/erc-20/>]. There is a unique account called the *Collar pool*, denoted $a_{\mathtt{pool}}$, which functions as the lending pool and is involved in every operation in the Collar core protocol. Any balance in the Collar pool $s_{c,a_{\mathtt{pool}}}$ is referred to as a *reserve*. 

$\mathbf{S}$ can also be expressed by the following shorthand notation (note $\mathbb{R}_\mathtt{0}$ is required because balance must always be a non-negative number):

$$
\mathbf{S} = (s_{c,a}) \in \mathbb{R}_\mathtt{0}^{|\mathbb{C}| \times |\mathbb{A}|} 
$$

The Collar core protocol can be represented as a finite state machine^[<https://en.wikipedia.org/wiki/Finite-state_machine>] with the input alphabet, state set, and initial state all being obvious, so that only the state transition function need be described here.

Let $\mathbf{S}_t$ denotes the state $\mathbf{S}$ at time $t$ with $t \in \mathbb{N}_{\mathtt{0}}$ and fix a constant expiry time $t_{\mathtt{expiry}}$. $t_{\mathtt{expiry}}$ will determine the actions available to a user interacting with the Collar core protocol at any time $t$: before $t_{\mathtt{expiry}}$, lenders and borrowers are able to perform lending and borrowing functions, while after $t_{\mathtt{expiry}}$, lenders are able to collect repaid tokens/underlying collateral. $f$ is used to denote the Collar operation and $g$ is used to denote a non-Collar operation. Details of $g$ will not be described in this paper because it is not related to the Collar core protocol, but $g$ should always meet some basic requirements (e.g., $g$ could be required to satisfy the ERC-20^[<https://ethereum.org/en/developers/docs/standards/tokens/erc-20/>] token standard). The state transition function shown below indicates the next state $\mathbf{S}_{t+1}$ can be either the previous state $\mathbf{S}_t$, a new state created by the Collar operation, or a new state determined by *g*, depending on which action is taken by the user:

$$
\mathbf{S}_{t+1} = \mathbf{S}_t \lor f(\mathbf{S}_t) \lor g(\mathbf{S}_t)
$$

$f$, the Collar operation, can be described as follows:

$$
f(\mathbf{S}_t) = (f_{\mathtt{-}}(\mathbf{S}_t) \land t < t_{\mathtt{expiry}}) \lor (f_{\mathtt{+}}(\mathbf{S}_t) \land t > t_{\mathtt{expiry}})
$$

where:

$$
f_{\mathtt{-}}(\mathbf{S}_t) \in \bigcup{\begin{Bmatrix}
    {f_{\mathtt{MintDual}}{(\mathbf{S}_t, a, n)}\: |\: a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
    {f_{\mathtt{MintColl}}{(\mathbf{S}_t, a, n)}\: |\: a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
    {f_{\mathtt{BurnDual}}{(\mathbf{S}_t, a, n)}\: |\: a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
    {f_{\mathtt{BurnCall}}{(\mathbf{S}_t, a, n)}\: |\: a \in \mathbb{A}, n \in \mathbb{R}_{\mathtt{0}}} \\
\end{Bmatrix}} \\
$$

$$
f_{\mathtt{+}}(\mathbf{S}_t) \in \left \{{f_{\mathtt{BurnColl}}{(\mathbf{S}_t, a, n)}\: |\: a \in \mathbb{A}, c \in \mathbb{C}}\right \}
$$

This means if it is before expiry time $t_{\mathtt{expiry}}$, then there are four operations (`MintDual`, `MintColl`, `BurnDual`, and `BurnCall`) that can be executed by account $a$ with a parameter $n \in \mathbb{R}_{\mathtt{0}}$, otherwise only the operation `BurnColl` can be executed.

Before describing the five operations (`MintDual`, `MintColl`, `BurnDual`, `BurnCall` and `BurnColl`), 3 auxillary operations will be defined first. In what follows, $\mathbf{J}^{c, a_{\mathtt{s}}}$ denotes a single-entry matrix^[<https://en.wikipedia.org/wiki/Single-entry_matrix>].

- **transfer**: $\mathbf{S}^{a_{\mathtt{s}} \overset{n}{\underset{{c}}{\longrightarrow}} a_{\mathtt{t}}}$ signifies an operation in which account $a_{\mathtt{s}}$ transfers amount $n$ of token $c$ to account $a_{\mathtt{t}}$:
    
    $$
    \mathbf{S}^{a_{\mathtt{s}} \overset{n}{\underset{{c}}{\longrightarrow}} a_{\mathtt{t}}} = \mathbf{S} - n \mathbf{J}^{c, a_{\mathtt{s}}} + n \mathbf{J}^{c, a_{\mathtt{t}}}
    $$
    

- **mint**: $\mathbf{S}^{\overset{n}{\underset{{c}}{\longrightarrow}} a}$ signifies an operation in which account $a$ receives amount $n$ of token $c$:
    
    $$
    \mathbf{S}^{\overset{n}{\underset{{c}}{\longrightarrow}} a} = \mathbf{S} + n \mathbf{J}^{c, a}
    $$

- **burn**: $\mathbf{S}^{a \overset{n}{\underset{{c}}    {\longrightarrow}}}$ signifies an operation in which account $a$ burns amount $n$ of token $c$:

    $$
    \mathbf{S}^{a \overset{n}{\underset{{c}}    {\longrightarrow}}} = \mathbf{S} - n \mathbf{J}^{c, a}
    $$

For simplicity, the composite operation $(\mathbf{S}^x)^y$ will also be denoted $\mathbf{S}^{x, y}$ and we extend this notation to any finite number of composed operations:

$$
\mathbf{S}^{x, y} = (\mathbf{S}^x)^y
$$
$$
\mathbf{S}^{x_{1}, x_{2},\cdots ,x_{n}} = (\cdots((\mathbf{S}^{x_{1}})^{x_{2}})\cdots)^{x_{n}}
$$

## MintDual

`MintDual` is the most important operation in the Collar core protocol. Using `MintDual`, if account $a$ deposits amount $n$ of $c_{\mathtt{bond}}$ tokens into the Collar pool $a_{\mathtt{pool}}$, then account $a$ will mint amount $n$ of both $c_{\mathtt{call}}$ tokens and $c_{\mathtt{coll}}$ tokens:

$$
f_{\mathtt{MintDual}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a_{\mathtt{pool}},  \overset{n}{\underset{c_{\mathtt{call}}}{\longrightarrow}} a,  \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}} a}
$$

Account $a$ can use this operation to borrow via the following mechanism. After the `MintDual` operation, account *a* can sell the received $c_{\mathtt{coll}}$ tokens for amount $n_{*}$ of $c_{\mathtt{want}}$ tokens. $n_{*}$ should be slightly less than $n$ because $n - n_{*}$ is the borrow interest. Later, account *a* can burn amount $n$ of $c_{\mathtt{call}}$ tokens and pay amount $n$ of $c_{\mathtt{want}}$ tokens to the Collar pool $a_{\mathtt{pool}}$ to receive its initial deposit amount $n$ of $c_{\mathtt{bond}}$ tokens back via the `BurnCall` operation before expiry time $t_{\mathtt{expiry}}$.

## MintColl

The `MintColl` operation is used for account $a$ to pay amount $n$ of $c_{\mathtt{want}}$ tokens to the Collar pool $a_{\mathtt{pool}}$ and directly receive amount $n$ of $c_{\mathtt{coll}}$ tokens:

$$
f_{\mathtt{MintColl}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{want}}}{\longrightarrow}} a_{\mathtt{pool}},  \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}} a}
$$

This operation can prevent shortages of $c_{\mathtt{coll}}$ tokens and keeps the price of $c_{\mathtt{coll}}$ tokens always slightly less than $c_{\mathtt{want}}$ tokens, otherwise arbitrageurs will swap between $c_{\mathtt{want}}$ tokens and $c_{\mathtt{coll}}$ tokens.

The `MintColl` operation is a composite of the `MintDual` operation and the `BurnCall` operation, and as such it functions just like an instant loan (borrow and then payback instantly):

$$
f_{\mathtt{MintColl}}{(\mathbf{S}, a, n)} = f_{\mathtt{BurnCall}}{(f_{\mathtt{MintDual}}{(\mathbf{S}, a, n)}, a, n)} 
$$

## BurnDual

A redeem operation `BurnDual` is provided for account $a$ to burn amount $n$ of both $c_{\mathtt{call}}$ tokens and $c_{\mathtt{coll}}$ tokens to get its locked $c_{\mathtt{bond}}$ tokens back:

$$
f_{\mathtt{BurnDual}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{call}}}{\longrightarrow}}, a \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}},a_{\mathtt{pool}} \overset{n}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a}
$$

Borrowers can use this operation to their benefit because the price of token $c_{\mathtt{coll}}$ should always be slightly less than token $c_{\mathtt{want}}$, and if the borrower repays much earlier than the expiry time $t_{\mathtt{expiry}}$, then it can sell $c_{\mathtt{want}}$ tokens for $c_{\mathtt{coll}}$ tokens and redeem them.

## BurnCall

`BurnCall` is the repay operation. Using `BurnCall`, account $a$ burns amount $n$ of $c_{\mathtt{call}}$ tokens and pays amount $n$ of $c_{\mathtt{want}}$ tokens to the Collar pool $a_{\mathtt{pool}}$, and then account *a* receives amount $n$ of $c_{\mathtt{bond}}$ tokens:

$$
f_{\mathtt{BurnCall}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{call}}}{\longrightarrow}}, a \overset{n}{\underset{c_{\mathtt{want}}}{\longrightarrow}} a_{\mathtt{pool}},a_{\mathtt{pool}} \overset{n}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a}
$$

## BurnColl

`BurnColl` is an operation provided for lenders. After expiry time $t_{\mathtt{expiry}}$, $c_{\mathtt{coll}}$ token holders can burn their $c_{\mathtt{coll}}$ tokens to receive the underlying $c_{\mathtt{bond}}$ tokens and $c_{\mathtt{want}}$ tokens from the Collar pool $a_{\mathtt{pool}}$:

$$
f_{\mathtt{BurnColl}}{(\mathbf{S}, a, n)} = \mathbf{S}^{a \overset{n}{\underset{c_{\mathtt{coll}}}{\longrightarrow}}, a_{\mathtt{pool}} \overset{\frac{n s_{c_{\mathtt{bond}}, a_{\mathtt{pool}}}}{\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}}}{\underset{c_{\mathtt{bond}}}{\longrightarrow}} a, a_{\mathtt{pool}} \overset{\frac{n s_{c_{\mathtt{want}}, a_{\mathtt{pool}}}}{\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}}}{\underset{c_{\mathtt{want}}}{\longrightarrow}} a }
$$ 

$\mathbf{J}^{1, c_{\mathtt{coll}}}$ is still the single entry matrix^[<https://en.wikipedia.org/wiki/Single-entry_matrix>] and $\mathbf{J}_{|\mathbb{A}|, 1}$ is a matrix of all ones^[<https://en.wikipedia.org/wiki/Matrix_of_ones>], thus, $\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}$ represents the total supply of token $c_{\mathtt{coll}}$. Then $\frac{n}{\mathbf{J}^{1, c_{\mathtt{coll}}} \mathbf{S} \mathbf{J}_{|\mathbb{A}|, 1}}$ is the total proportion of the $c_{\mathtt{coll}}$ burned by the operation and the lender receives the same proportion of both the $c_{\mathtt{bond}}$ and $c_{\mathtt{want}}$ reserves.


# Analysis

## Call Token

The token $c_{\mathtt{call}}$ is the right of repayment for borrowers before $t_{\mathtt{expiry}}$. It is just like buying a call option for token $c_{\mathtt{bond}}$.

Thus, after $t_{\mathtt{expiry}}$ the $c_{\mathtt{call}}$ token will be useless and before $t_{\mathtt{expiry}}$ the $c_{\mathtt{call}}$ token is insurance for both the $c_{\mathtt{bond}}$ token and $c_{\mathtt{want}}$ token. This is because the $c_{\mathtt{call}}$ token gives its holder the right to choose the higher priced one between $c_{\mathtt{bond}}$ and $c_{\mathtt{want}}$. Keeping these facts in mind, it's clear that if both the $c_{\mathtt{bond}}$ and $c_{\mathtt{want}}$ tokens remain pegged, then:

* the price of $c_{\mathtt{call}}$ should always be close to 0.
* the price of $c_{\mathtt{call}}$ should be greater for $c_{\mathtt{bond}}$/$c_{\mathtt{want}}$ pairs with low confidence in the pegs than in pairs with high confidence.
* the price of $c_{\mathtt{call}}$ should decrease over time until reaching 0 at $t_{\mathtt{expiry}}$.

If the $c_{\mathtt{bond}}$ and $c_{\mathtt{want}}$ do not both remain pegged, then at $t_{\mathtt{expiry}}$:

* if $\textrm{price of }c_{\mathtt{bond}} < \textrm{price of }c_{\mathtt{want}}$, then $\textrm{price of }c_{\mathtt{call}} = 0$.
* if $\textrm{price of }c_{\mathtt{bond}} > \textrm{price of }c_{\mathtt{want}}$, then $\textrm{price ratio }\frac{c_{\mathtt{call}}}{c_{\mathtt{bond}}}\: +\: \textrm{price ratio }\frac{c_{\mathtt{want}}}{c_{\mathtt{bond}}} = 1$, otherwise an arbitrage opportunity exists.

## Coll Token

The token $c_{\mathtt{coll}}$ is the right of liquidation for lenders after $t_{\mathtt{expiry}}$. It is just like writing a put option for token $c_{\mathtt{bond}}$.

Before $t_{\mathtt{expiry}}$, the price of token $c_{\mathtt{coll}}$ acts as the borrow interest. If the $c_{\mathtt{bond}}$ and $c_{\mathtt{want}}$ tokens remain pegged, then:

* the price of $c_{\mathtt{coll}}$ should always be slightly less than the price of $c_{\mathtt{want}}$.
* the price of $c_{\mathtt{coll}}$ should increase over time until reaching the price of $c_{\mathtt{want}}$ at $t_{\mathtt{expiry}}$.   

If the $c_{\mathtt{bond}}$ and $c_{\mathtt{want}}$ tokens do not both remain pegged, then at $t_{\mathtt{expiry}}$:

* if $\textrm{price of }c_{\mathtt{bond}} < \textrm{price of }c_{\mathtt{want}}$, then $\textrm{price ratio }\frac{c_{\mathtt{coll}}}{c_{\mathtt{want}}} = \textrm{price ratio }\frac{c_{\mathtt{bond}}}{c_{\mathtt{want}}}$,
* if $\textrm{price of }c_{\mathtt{bond}} > \textrm{price of }c_{\mathtt{want}}$, then $\textrm{price ratio }\frac{c_{\mathtt{coll}}}{c_{\mathtt{want}}} = 1$,

otherwise an arbitrage opportunity exists.

There are several market forces that keep the price ratio $\frac{c_{\mathtt{coll}}}{c_{\mathtt{want}}}$ in check. Firstly, if $\textrm{price ratio }\frac{c_{\mathtt{coll}}}{c_{\mathtt{want}}} > 1$, then this can be arbitraged by using `MintColl` to swap between $c_{\mathtt{coll}}$ and $c_{\mathtt{want}}$ as previously mentioned. If $\textrm{price ratio }\frac{c_{\mathtt{coll}}}{c_{\mathtt{want}}} \ll 1$, then:

* lenders will take advantage of the high interest rate and buy more $c_{\mathtt{coll}}$.
* borrowers holding $c_{\mathtt{call}}$ and $c_{\mathtt{want}}$ tokens will take advantage of the interest rate being higher than when they borrowed and buy $c_{\mathtt{coll}}$ to use to redeem using `BurnDual` and receive their $c_{\mathtt{bond}}$ tokens.  

# Summary


* Collar is a lending protocol that creates money markets. It does this via a decentralized mechanism that is able to have an elegant, simple design due to the expectation of the protocol primarily being used for lending and borrowing pegged assets.
* Collar acts as insurance for borrowers from de-pegs for both the asset they borrow and the one they deposit as collateral (though not both at the same time).
* Borrowers pay a fixed interest rate and their collateral cannot be liquidated until after the expiry time. 
* Lenders earn a fixed, predictable interest rate and can withdraw their assets at any time.
* After the expiry time, lenders receive both the repaid and collateral assets in proportions determined by the amounts repaid and defaulted by borrowers. 

\ 

\ 

\ 


# Appendix

## Notation

| symbol                                                                       | description                                   |
| ---------------------------------------------------------------------------- | --------------------------------------------- |
| $a$                                                                          | account (blockchain wallet address)           |
| $a_{\mathtt{pool}}$                                                          | Collar pool account                           |
| $\mathbb{A}$                                                                 | set of all accounts                           |
| $c$                                                                          | token (fungible token)                        |
| $c_{\mathtt{bond}}$                                                          | bond token                                    |
| $c_{\mathtt{call}}$                                                          | call token                                    |
| $c_{\mathtt{coll}}$                                                          | coll token                                    |
| $c_{\mathtt{want}}$                                                          | want token                                    |
| $\mathbb{C}$                                                                 | set of all types of token                     |
| $f_{\mathtt{BurnCall}}$                                                      | `BurnCall` operation                          |
| $f_{\mathtt{BurnColl}}$                                                      | `BurnColl` operation                          |
| $f_{\mathtt{BurnDual}}$                                                      | `BurnDual` operation                          |
| $f_{\mathtt{MintColl}}$                                                      | `MintColl` operation                          |
| $f_{\mathtt{MintDual}}$                                                      | `MintDual` operation                          |
| $\mathbf{J}^{i,j}$                                                           | single entry matrix                           |
| $\mathbf{J}_{m,n}$                                                           | $m \times n$ dimension matrix of all ones     |
| $n$                                                                          | amount                                        |
| $\mathbb{N}_\mathtt{0}$                                                      | set of all non-negative integers              |
| $\mathbb{R}_\mathtt{0}$                                                      | set of all non-negative real numbers          |
| $s_{c, a}$                                                                   | balance of token $c$ of account $a$           |
| $\mathbf{S}$                                                                 | matrix of all $s_{c, a}$                      |
| $a_{\mathtt{s}} \overset{n}{\underset{{c}}{\longrightarrow}} a_{\mathtt{t}}$ | transfer operation                            |
| $\overset{n}{\underset{{c}}{\longrightarrow}} a$                             | mint operation                                |
| $a \overset{n}{\underset{{c}}{\longrightarrow}}$                             | burn operation                                |
