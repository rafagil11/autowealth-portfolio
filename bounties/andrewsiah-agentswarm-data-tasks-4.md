# Bounty Deliverable: Parse bounty reward strings for the feed importer
- **Target Repository**: `andrewsiah/agentswarm-data-tasks` (Issue #4)
- **Upstream Issue**: https://github.com/andrewsiah/agentswarm-data-tasks/issues/4
- **Reward**: **$450.0 USD**
- **Autonomous GitHub Status**: `BRANCH_COMMITTED_ON_GITHUB`
- **Archivos Subidos al Fork**: `tasks/reward-parser/reward_parser.py`
- **Forked Branch / Commit URL**: https://github.com/rafagil11/agentswarm-data-tasks/tree/fix/issue-4
- **Pull Request / 1-Click Compare URL**: https://github.com/andrewsiah/agentswarm-data-tasks/compare/main...rafagil11:agentswarm-data-tasks:fix/issue-4?expand=1

---

### Summary
- Implemented `parse_reward(text)` in `tasks/reward-parser/reward_parser.py` conforming to `SPEC.md`.
- Supports robust parsing of various reward strings containing amounts, currencies (cash, crypto, points, site-balance), modifiers, prefixes, and suffixes.

### Root Cause
- `parse_reward` was previously unimplemented (`NotImplementedError`), causing reward parsing checks to fail.

### Technical Changes
- Developed a comprehensive regex-based parser that validates and extracts amounts (as exact `decimal.Decimal`), currency symbols/tokens, cash/crypto/points/site-balance categories, and handles optional prefixes (`reward:`, `bounty:`, `up to`, `approximately`, `~`) and suffixes (`site balance`, `gross of site balance`, `(test fixture)`).
- Ensured strict constraint adherence (no float conversion, precise decimal handling, case-insensitivity, whitespace normalization, and safe fallback to unknown/None for unsupported cases like `Unpaid`).

### Verification/Tests
- Verified successfully against all authored synthetic cases and generated value/case/whitespace probes via `python3 tasks/reward-parser/check.py`.

---
Closes #4


---

## 💻 Código Fuente Subido a la Rama (`https://github.com/rafagil11/agentswarm-data-tasks/tree/fix/issue-4`)

### Archivo Modificado/Creado: `tasks/reward-parser/reward_parser.py`
```py
from decimal import Decimal, InvalidOperation
import re


def parse_reward(text: str) -> dict:
    if not isinstance(text, str):
        return {"amount": None, "currency": None, "kind": "unknown"}

    s = text.strip()

    prefix_pattern = re.compile(r'^(?:reward|bounty)\s*:\s*', re.IGNORECASE)
    s = prefix_pattern.sub('', s).strip()

    qualifier_pattern = re.compile(r'^(?:up\s+to|approximately|~)\s*', re.IGNORECASE)
    s = qualifier_pattern.sub('', s).strip()

    test_fixture_pattern = re.compile(r'\s*\(\s*test\s+fixture\s*\)\s*$', re.IGNORECASE)
    s = test_fixture_pattern.sub('', s).strip()

    site_balance_override = False
    sb_patterns = [
        re.compile(r'\s+gross\s+of\s+site\s+balance\s*$', re.IGNORECASE),
        re.compile(r'\s+\(?\s*site\s+balance\s*\)?\s*$', re.IGNORECASE),
    ]
    for pat in sb_patterns:
        if pat.search(s):
            site_balance_override = True
            s = pat.sub('', s).strip()
            break

    tokens = s.split()
    if not tokens or len(tokens) > 2:
        return {"amount": None, "currency": None, "kind": "unknown"}

    amount_str = None
    curr_str = None
    symbol_found = None

    def parse_amount_str(t):
        sym = None
        if t.startswith(('$', '€', '£')):
            sym = t[0]
            t = t[1:]
        amt_pat = re.compile(r'^(?:\d+|\d{1,3}(?:,\d{3})+)(?:\.\d+)?$')
        if amt_pat.match(t):
            return t, sym
        return None, None

    if len(tokens) == 1:
        t = tokens[0]
        if t[0] in ('$', '€', '£'):
            sym = t[0]
            rem = t[1:]
            amt, _ = parse_amount_str(rem)
            if amt:
                amount_str = amt
                symbol_found = sym
                if sym == '$':
                    curr_str = 'USD'
                elif sym == '€':
                    curr_str = 'EUR'
                elif sym == '£':
                    curr_str = 'GBP'
            else:
                return {"amount": None, "currency": None, "kind": "unknown"}
        else:
            return {
                "amount": None,
                "currency": None,
                "kind": "unknown",
            }
    elif len(tokens) == 2:
        t1, t2 = tokens[0], tokens[1]

        amt1, sym1 = parse_amount_str(t1)
        if amt1:
            amount_str = amt1
            symbol_found = sym1
            curr_str = t2
        else:
            amt2, sym2 = parse_amount_str(t2)
            if amt2:
                amount_str = amt2
                symbol_found = sym2
                curr_str = t1
            else:
                return {
                    "amount": None,
                    "currency": None,
                    "kind": "unknown",
                }

    if not amount_str or not curr_str:
        return {"amount": None, "currency": None, "kind": "unknown"}

    curr_upper = curr_str.upper()
    if curr_upper == 'POINT':
        curr_upper = 'POINTS'
    elif curr_upper == 'CREDIT':
        curr_upper = 'CREDITS'

    if symbol_found:
        if symbol_found == '$':
            if curr_upper not in ('USD', 'USDC', 'USDT'):
                return {
                    "amount": None,
                    "currency": None,
                    "kind": "unknown",
                }
        elif symbol_found == '€':
            if curr_upper != 'EUR':
                return {
                    "amount": None,
                    "currency": None,
                    "kind": "unknown",
                }
        elif symbol_found == '£':
            if curr_upper != 'GBP':
                return {
                    "amount": None,
                    "currency": None,
                    "kind": "unknown",
                }

    cash_currencies = {'USD', 'EUR', 'GBP', 'CAD', 'AUD', 'CHF', 'JPY', 'INR'}
    crypto_currencies = {'USDC', 'USDT', 'ETH', 'BTC', 'SOL'}
    points_currencies = {'XTR', 'POINTS'}
    site_balance_currencies = {'CREDITS'}

    kind = None
    if curr_upper in cash_currencies:
        kind = 'cash'
    elif curr_upper in crypto_currencies:
        kind = 'crypto'
    elif curr_upper in points_currencies:
        kind = 'points'
    elif curr_upper in site_balance_currencies:
        kind = 'site-balance'
    else:
        return {"amount": None, "currency": None, "kind": "unknown"}

    if site_balance_override:
        kind = 'site-balance'

    try:
        clean_amt_str = amount_str.replace(',', '')
        dec_amount = Decimal(clean_amt_str)
        if not dec_amount.is_finite() or dec_amount < 0:
            return {"amount": None, "currency": None, "kind": "unknown"}
    except (InvalidOperation, ValueError):
        return {"amount": None, "currency": None, "kind": "unknown"}

    return {
        "amount": dec_amount,
        "currency": curr_upper,
        "kind": kind,
    }

```
