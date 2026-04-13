# Suggested Tips Flow — QSR vs TST

```mermaid
flowchart TD
    START([Order Ready for Payment]) --> CHECK{IS_QUICK_SERVE?}

    %% ─────────────────────────────
    %% QSR BRANCH
    %% ─────────────────────────────
    CHECK -- QSR --> QA[calculatePercentagesFromDb\nBuild 3 TipDisplayItems\nfallback: 18% / 20% / 22%]

    QA --> QB{Payment Method}

    QB -- Card --> QC[SecondaryScreenUiState\nShowPayByCardTip\nPayByCardTipSecondaryScreen]
    QB -- Tender/Cash --> QD[SecondaryScreenUiState\nShowPayByTenderTip\nPayByTenderTip]
    QB -- Gift Card --> QE[ShowPayByGiftTipScreen]

    QC --> QF[Customer sees 3 suggested\ntip buttons + Custom Tip\nDefault pre-highlighted]
    QD --> QF
    QE --> QF

    QF --> QG{Customer selects}
    QG -- Suggested % --> QH[onTipSelect callback\ntipAmount returned]
    QG -- Custom --> QI[Numpad input]
    QG -- No Tip --> QJ[tipAmount = 0]
    QI --> QH

    QH --> QK{Tip > 50% of total?}
    QK -- Yes --> QL[Confirmation Dialog]
    QL -- Confirmed --> QM
    QL -- Cancel --> QF
    QK -- No --> QM

    QM[Total = Order + Tip\nAnalytics: postPaymentTipEntered]
    QM --> QN[Payment processed\nwith tip included]
    QN --> QO[printReceipt server\nReceipt shows actual tip paid]
    QO --> QEND([QSR Done])

    %% ─────────────────────────────
    %% TST BRANCH
    %% ─────────────────────────────
    CHECK -- TST --> TA[Payment processed\nwith tip = 0.00]
    TA --> TB{tip == 0.00?\nwillTipAddLater}
    TB -- Yes --> TC[printReceiptLaterTip\nprinterViewModel\n.invokePayByCardWithTip]
    TB -- No --> TD[printReceipt server\nnormal receipt]

    TC --> TE[Receipt printed with\ntip1 / tip2 / tip3\nsuggested percentages]
    TE --> TF[Server presents\nreceipt to customer]
    TF --> TG{Customer writes\ntip on receipt?}
    TG -- Yes --> TH[Server adds tip later\nvia AddTipSelectionDialog\nUnclosed Tickets tab]
    TG -- No Tip --> TI([TST Done - No Tip])

    TH --> TJ[AddTipUnclosedFragment\nAddTipUnclosedNumPadView]
    TJ --> TK{Tip > 50%?}
    TK -- Yes --> TL[Warning shown\nisTipAbove50Added = true]
    TK -- No --> TM
    TL --> TM[updateTipAmount on adapter\nTicket updated]
    TM --> TEND([TST Done - Tip Added])

    %% ─────────────────────────────
    %% STYLING
    %% ─────────────────────────────
    style START fill:#4CAF50,color:#fff
    style CHECK fill:#1565C0,color:#fff
    style QEND fill:#2196F3,color:#fff
    style TEND fill:#2196F3,color:#fff
    style TI fill:#9E9E9E,color:#fff
    style QJ fill:#9E9E9E,color:#fff
    style QL fill:#FF9800,color:#fff
    style TL fill:#FF9800,color:#fff
    style QF fill:#E3F2FD,color:#000
    style TE fill:#FFF9C4,color:#000
```

## Difference at a Glance

| | QSR | TST |
|---|---|---|
| **When tips shown** | On secondary screen during payment | On printed receipt after payment |
| **Who selects tip** | Customer (at POS screen) | Customer (handwritten on receipt) |
| **Tip in payment request** | Included immediately | `willTipAddLater = true`, added later |
| **Receipt print call** | `printReceipt("server")` | `printReceiptLaterTip()` |
| **Receipt content** | Actual tip amount paid | Suggested tip % lines (tip1/tip2/tip3) |
| **Post-payment tip entry** | Not needed | `AddTipUnclosedFragment` via dialog |
| **Key flag** | `IS_QUICK_SERVE = true` | `IS_QUICK_SERVE = false` |

## Key Files

| File | Line | Role |
|------|------|------|
| `Constants.kt` | 89 | `IS_QUICK_SERVE` flag |
| `PayByCardFragment.kt` | 1461 | QSR: trigger `ShowPayByCardTip` secondary screen |
| `PayByCardFragment.kt` | 1794 | TST: set `willTipAddLater` flag |
| `PayByCardFragment.kt` | 614 | Branch: `printReceiptLaterTip()` vs `printReceipt()` |
| `Extensions.kt` | 3599 | `calculatePercentagesFromDb()` — builds tip options |
| `PayByCardTipSecondaryScreen.kt` | — | QSR card tip selection UI |
| `PayByTenderTip.kt` | — | QSR tender tip selection UI |
| `PrinterTemplates.kt` | 2034 | `printServerTips()`, tip receipt lines |
| `AddTipUnclosedFragment.kt` | — | TST: add tip to unclosed ticket |
```
