---
tags:
  - options
  - trading
  - notation
  - reference
---

# Options Trading Notation System

A unified notation system for defining, tracking, and analyzing options positions across their full lifecycle — from open through rolls to close. Designed to be used in strategy documents, trade journals, and code.

> [!note] LaTeX Notation
> Superscripts denote roll state ($O^1$, $O^2$, $O^n$) or special state ($O^r$ for candidate). Subscripts denote attributes of the position or leg.

---

## Conventions

| Convention | Meaning |
|---|---|
| Superscripts | Roll state ($O^1, O^2, O^n$) or special state ($O^r$ for candidate) |
| Subscripts | Attributes of the position or leg |
| Sign on value | $+$ = credit received · $-$ = debit paid. The symbol itself carries no sign. |
| `$` prefix on symbol | Denotes a monetary value (credit or debit) |
| `C` prefix on symbol | Denotes a closing transaction |

---

## Account Symbols

### Account Balances

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Av` | $A_v$ | Account liquid value (total) |
| `Ac` | $A_c$ | Account cash balance |
| `Ao` | $A_o$ | Capital held up by all open positions $\left(A_o = \sum O_c\right)$ |
| `Ar` | $A_r$ | Capital available to deploy $\left(A_r = A_c - A_o\right)$ |

---

## Position State Symbols

### Position Lifecycle

| Symbol | LaTeX | Meaning |
|---|---|---|
| `O` | $O$ | Original option position (before any rolls); equivalent to $O^0$ |
| `O⁰` | $O^0$ | Original position (explicit form, interchangeable with $O$) |
| `Oⁿ` | $O^n$ | Position state after $n$th roll; contains full set of active legs |
| `Oʳ` | $O^r$ | Rolling candidate — proposed next state under evaluation; becomes $O^{n+1}$ if accepted |
| `OC` | $OC$ | Final close of the entire position chain |

---

## Capital & Fees

### Buying Power & Fees

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Oc` | $O_c$ | Capital held up (buying power reduction) by original position |
| `Oⁿc` | $O^n_c$ | Capital held up after $n$th roll |
| `Oʳc` | $O^r_c$ | Capital held up by rolling candidate |
| `Of` | $O_f$ | Fee paid to open original position |
| `Oⁿf` | $O^n_f$ | Fee paid at $n$th roll |
| `Oʳf` | $O^r_f$ | Fee to execute the candidate roll |
| `OCf` | $OC_f$ | Fee paid at final close |

---

## Days to Expiry (DTE)

| Symbol | LaTeX | Meaning |
|---|---|---|
| `DTE` | $\text{DTE}$ | Days to expiry (nearest leg, real-time) |
| `O_DTE` | $O_{DTE}$ | DTE of original position |
| `Oⁿ_DTE` | $O^n_{DTE}$ | DTE after $n$th roll |
| `Oʳ_DTE` | $O^r_{DTE}$ | DTE of rolling candidate |
| `OC_DTE` | $OC_{DTE}$ | DTE at time of final close |

---

## Profit & Loss Targets

### Original Position

| Symbol | LaTeX | Meaning |
|---|---|---|
| `O_p25` | $O_{p25}$ | 25% of max profit on original position |
| `O_p50` | $O_{p50}$ | 50% of max profit on original position |
| `O_p75` | $O_{p75}$ | 75% of max profit on original position |
| `O_p100` | $O_{p100}$ | 100% of max profit on original position |
| `O_l25` | $O_{l25}$ | 25% of max loss on original position |
| `O_l50` | $O_{l50}$ | 50% of max loss on original position |
| `O_l75` | $O_{l75}$ | 75% of max loss on original position |
| `O_l100` | $O_{l100}$ | 100% of max loss on original position |

### After nth Roll & Candidate

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Oⁿ_p50` | $O^n_{p50}$ | 50% of max profit after $n$th roll |
| `Oⁿ_l50` | $O^n_{l50}$ | 50% of max loss after $n$th roll |
| `Oʳ_p50` | $O^r_{p50}$ | 50% of max profit of rolling candidate |
| `Oʳ_l50` | $O^r_{l50}$ | 50% of max loss of rolling candidate |

### Final Close

| Symbol | LaTeX | Meaning |
|---|---|---|
| `OC_p` | $OC_p$ | Final realized profit at close |
| `OC_l` | $OC_l$ | Final realized loss at close |

---

## Credits & Debits

### Open / Roll / Close Values

| Symbol | LaTeX | Meaning |
|---|---|---|
| `$O` | $\$O$ | Credit/debit received when opening original position |
| `$Oⁿ` | $\$O^n$ | Credit/debit at $n$th roll |
| `$Oʳ` | $\$O^r$ | Credit/debit to open rolling candidate |
| `$OC` | $\$OC$ | Credit/debit at final close |

---

## Closing Transaction Targets

| Symbol | LaTeX | Meaning |
|---|---|---|
| `CO_p50` | $CO_{p50}$ | Closing credit/debit required to realize 50% profit on original |
| `CO_p75` | $CO_{p75}$ | Closing credit/debit required to realize 75% profit on original |
| `COⁿ_p50` | $CO^n_{p50}$ | Closing credit/debit to realize 50% profit at $n$th roll |
| `COⁿ_p75` | $CO^n_{p75}$ | Closing credit/debit to realize 75% profit at $n$th roll |
| `COʳ_p50` | $CO^r_{p50}$ | Closing credit/debit to realize 50% profit on candidate |

---

## Underlying Price

| Symbol | LaTeX | Meaning |
|---|---|---|
| `U` | $U$ | Current underlying price (real-time) |
| `U₀` | $U_0$ | Underlying price at position open |
| `Uⁿ` | $U^n$ | Underlying price at $n$th roll |

---

## Leg Symbols

### Original Legs

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Pso` | $P_{so}$ | Put sold to open |
| `Pbo` | $P_{bo}$ | Put bought to open |
| `Cso` | $C_{so}$ | Call sold to open |
| `Cbo` | $C_{bo}$ | Call bought to open |

### Legs After nth Roll

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Psoⁿ` | $P^n_{so}$ | Put sold leg after $n$th roll |
| `Pboⁿ` | $P^n_{bo}$ | Put bought leg after $n$th roll |
| `Csoⁿ` | $C^n_{so}$ | Call sold leg after $n$th roll |
| `Cboⁿ` | $C^n_{bo}$ | Call bought leg after $n$th roll |

### Rolling Candidate Legs

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Psoʳ` | $P^r_{so}$ | Put sold leg of candidate |
| `Pboʳ` | $P^r_{bo}$ | Put bought leg of candidate |
| `Csoʳ` | $C^r_{so}$ | Call sold leg of candidate |
| `Cboʳ` | $C^r_{bo}$ | Call bought leg of candidate |

---

## Rolling Candidate — Full Reference

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Oʳ` | $O^r$ | Rolling candidate position |
| `Oʳ_c` | $O^r_c$ | Capital requirement |
| `Oʳ_f` | $O^r_f$ | Fee to execute |
| `Oʳ_DTE` | $O^r_{DTE}$ | Days to expiry |
| `Oʳ_p50` | $O^r_{p50}$ | 50% of max profit |
| `Oʳ_l50` | $O^r_{l50}$ | 50% of max loss |
| `$Oʳ` | $\$O^r$ | Credit/debit to open |
| `COʳ_p50` | $CO^r_{p50}$ | Closing credit/debit to achieve 50% profit |
| `Pʳso` | $P^r_{so}$ | Put sold leg |
| `Pʳbo` | $P^r_{bo}$ | Put bought leg |
| `Cʳso` | $C^r_{so}$ | Call sold leg |
| `Cʳbo` | $C^r_{bo}$ | Call bought leg |

---

## Account Formulas

| Symbol | LaTeX | Meaning |
|---|---|---|
| `Av` | $A_v$ | Account liquid value |
| `Ac` | $A_c$ | Account cash balance |
| `Ao` | $A_o$ | Capital held up by all open positions: $\displaystyle A_o = \sum O_c$ |
| `Ar` | $A_r$ | Capital available to deploy: $A_r = A_c - A_o$ |

---

## Position Lifecycle — At a Glance

| Symbol | LaTeX | Stage |
|---|---|---|
| `O` | $O$ | Original position |
| `Oⁿ` | $O^n$ | Position after $n$th roll |
| `Oʳ` | $O^r$ | Rolling candidate |
| `OC` | $OC$ | Final close |

---

## Example: Put Credit Spread Lifecycle

Opening a put credit spread on ORCL, rolling once, then closing at 50% profit.

### Open — $O$

| Field | Value |
|---|---|
| Legs | $P_{so}$: ORCL 160P exp Jun · $P_{bo}$: ORCL 155P exp Jun |
| $\$O$ | +$3.50 credit |
| $O_c$ | $500 buying power held |
| $O_{DTE}$ | 30 days |
| $O_{p50}$ | +$175 (50% of max profit) |

### Evaluate Roll Candidate — $O^r$ (at DTE = 10)

| Field | Value |
|---|---|
| Legs | $P^r_{so}$: ORCL 158P exp Jul · $P^r_{bo}$: ORCL 153P exp Jul |
| $\$O^r$ | +$2.80 credit |
| $O^r_c$ | $500 buying power held |
| $O^r_{DTE}$ | 38 days |
| $O^r_{p50}$ | +$140 (50% of max profit) |

### Promote Candidate → First Roll — $O^1 = O^r$

| Field | Value |
|---|---|
| $\$O^1$ | +$2.80 credit |
| $O^1_{DTE}$ | 38 days |

### Close at 50% Profit

| Field | Value |
|---|---|
| $CO^1_{p50}$ | −$1.40 debit (closing order) |
| $OC_{DTE}$ | 20 days remaining |
| $OC_p$ | +$140 realized profit |

### Account Summary

$$A_o = O^1_c = \$500 \text{ buying power occupied}$$

$$A_r = A_c - A_o$$
