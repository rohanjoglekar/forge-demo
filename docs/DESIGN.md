# Design — data integrity as an interface requirement

Forge's dashboard informs model-promotion and execution decisions; its presentation rules are therefore extensions of the research methodology. The interface must preserve measurement scope, calibrated precision, and operational risk without selectively emphasizing favorable outcomes. This document defines the principal rules and the incidents that established them.

## Every metric identifies its measurement window

A performance statistic without a defined time window changes meaning whenever a counter resets or a strategy is replaced. Forge makes the measurement start date part of the data contract. Kalshi performance is calculated from the activation date of the current strategy version, the backend returns that date with each result, and the interface labels every dependent figure with an explicit qualifier such as *since Jul 11*. Because 51% of the account's lifetime orders predate the current strategy, an unlabeled lifetime statistic would combine materially different systems.

This rule exposed a real defect in the public demonstration. A stale component rendered **“P/L vs $1,000: +$191.11” beside $967 of equity**. Both values were individually correct, but their juxtaposition was false: the +$191.11 covered only the current strategy version, while $1,000 was the account's lifetime opening balance. The corrected card measures P&L from the $776.21 of equity present when the current strategy began. Every displayed balance relationship must now reconcile arithmetically from the values shown.

## Trading modes use one risk vocabulary

Early dashboard sections described equivalent states inconsistently: “LIVE · Alpaca paper” appeared in green, “PAPER · EXECUTING” appeared in red, and “live · disarmed” appeared in lowercase. Forge now expresses operating state through two explicit dimensions: capital mode (`LIVE / PAPER / SHADOW`) and system behavior (`EXECUTING / ADVISORY / DISARMED`).

Color encodes financial risk, not activity or success. A service executing with real capital is always red; green is reserved for states that cannot spend money. A label such as “LIVE” may describe current data, but it is never styled in a way that could be mistaken for a safety state.

## Calibrated values retain calibrated precision

The measured tail half-spread is 4.25¢. Rendering it as 4.3¢ changes a calibrated input and creates an apparent discrepancy between the interface and the underlying configuration. The cents formatter therefore preserves meaningful decimal places and removes only trailing zeros.

The same policy applies throughout the dashboard. Raw binary floating-point artifacts never reach the interface; an earlier order-book table once displayed `0.4200000000000001`, leading to a rule that all prices pass through domain-specific formatting. Strategy identifiers may be abbreviated in summary views only when the complete configuration remains accessible in one interaction.

## Logs prioritize the signal required by their audience

The shadow-testing log records every live, order-free prediction. One ensemble strategy frequently records `PASS`, indicating that no position should be opened, and those events once represented approximately 80% of the raw public log. For an operator, pass frequency is a monitoring signal; for a public reader, it obscures executed decisions.

The private dashboard therefore defaults to the complete event stream, while the public demonstration defaults to trade signals and provides a one-click control for revealing all predictions. Both surfaces use the same underlying data and explicitly identify any active filter.

## Analytical tables must expose their conclusions without horizontal discovery

At common laptop width, the shadow-testing table's win-rate, z-statistic, and P&L columns were positioned approximately 300 pixels beyond the viewport without a visible indication that horizontal scrolling was available. Removing a redundant field, wrapping strategy identifiers, and introducing a denser spacing mode reduced the 11-column table from 1,699 pixels to 1,069 pixels. The columns that determine promotion eligibility are now visible at the screen width used for operational review.

## Static context must survive live-data failure

The public landing page renders its explanatory content without network requests. Live elements, including the three books' P&L and summary statistics, remain absent until the data request succeeds; they do not produce blocking spinners, error cards, or layout movement. A transient backend failure can therefore suppress current numbers, but it cannot prevent a visitor from understanding what Forge is or how it operates.
