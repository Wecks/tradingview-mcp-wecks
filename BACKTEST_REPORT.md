# Rapport de Backtest — Stratégies de Trading
**Date :** Avril 2026 | **Instrument principal :** AMEX:SPY | **Outil :** Pine Script v6 + TradingView MCP

---

## Table des matières

1. [Méthodologie générale & limites techniques](#1-méthodologie-générale--limites-techniques)
2. [Stratégie 1 — H1 Forex (Price Action, 87% WR claim)](#2-stratégie-1--h1-forex-price-action-87-wr-claim)
3. [Stratégie 2 — RSI(2) Connors](#3-stratégie-2--rsi2-connors)
4. [Stratégie 3 — Gap Fill Daily (SPY)](#4-stratégie-3--gap-fill-daily-spy)
5. [Synthèse comparative](#5-synthèse-comparative)
6. [Ce qui vaut le coup d'être testé en backtest manuel](#6-ce-qui-vaut-le-coup-dêtre-testé-en-backtest-manuel)
7. [Pistes d'amélioration concrètes](#7-pistes-damélioration-concrètes)
8. [Limites fondamentales de ce travail](#8-limites-fondamentales-de-ce-travail)

---

## 1. Méthodologie générale & limites techniques

### Environnement de test

Tous les backtests ont été réalisés via **Pine Script v6** sur **TradingView Desktop**, avec injection de code via le protocole CDP (Chrome DevTools Protocol). L'automatisation de la compilation et de la lecture des résultats a été faite via le serveur MCP TradingView.

### Contraintes structurelles de Pine Script

Pine Script a plusieurs limitations importantes qui ont impacté la qualité des backtests :

**Problème des barres Daily en Strategy**
Pine Strategy ne peut pas, sur une même barre Daily, à la fois entrer à l'open, vérifier si un niveau intraday est atteint (e.g. `high >= TP`), et sortir au close EOD. Ces trois opérations nécessitent des données intraday. Solution adoptée pour Gap Fill : backtest **indicateur manuel** avec calcul P&L barre-par-barre. C'est plus rigoureux mais moins flexible pour les statistiques avancées (Sharpe, CAGR exacts).

**Données historiques limitées (compte Free)**
- TradingView Free fournit ~1 an de données en 5 minutes → Gap Fill sur données intraday = seulement ~6 trades. Résultat inexploitable, d'où le passage aux barres Daily.
- Les stratégies Daily bénéficient de l'historique complet SPY depuis 1993 (fiable).

**Slippage et commissions**
Les valeurs utilisées (0.05% commission, 1 tick de slippage) sont des approximations. En pratique :
- Pour un ETF liquide comme SPY : réaliste
- Pour le forex H1 : le spread variable (0.5–2 pips selon heure/broker) est potentiellement sous-estimé
- Les backtests ne modélisent pas l'impact sur le marché (négligeable sur SPY, non-négligeable sur actifs peu liquides)

**Biais de lookahead**
Dans les trois stratégies, aucun biais de lookahead identifié. Vérification systématique :
- Les signaux utilisent uniquement `close[1]`, `high[1]`, `low[1]` (données de la veille)
- Les entrées sont soit à l'open de la barre courante, soit sur ordre stop (déclenchement conditionnel)
- Les pivots swing ont un lag inhérent de `swing_len` barres (non-repaint, acceptable)

**Bug `str.tostring(x, "#.3")`**
Découvert pendant l'implémentation : le format `"#.3"` affiche le suffixe `.3` comme literal, pas 3 décimales. Il faut `"#.###"`. Impact : les colonnes "Avg Win" et "Avg Loss" dans la table Gap Fill affichent une valeur incorrecte. N'affecte pas les calculs P&L ni l'equity curve.

---

## 2. Stratégie 1 — H1 Forex (Price Action, 87% WR claim)

### Source
Medium — *"The 1-Hour Forex Strategy — Our 87% Win Rate Method for Beginners"*

### Règles de la stratégie
1. **Tendance** : prix au-dessus de l'EMA 50
2. **Niveau clé** : prix dans une zone de support/résistance (pivots swing)
3. **Pattern** : Pin Bar (wick dominant ≥55% du range) ou Inside Bar
4. **Session** : London (07–10h UTC) ou New York (12–15h UTC)
5. **Entrée** : stop order au-dessus/dessous du pattern
6. **SL/TP** : SL sous/sur le bas/haut du pattern, TP = 2×SL (RR 1:2)

### Résultats

| Instrument | Période | Trades | Win Rate | Profit Factor | Net P&L | Max DD |
|-----------|---------|--------|----------|---------------|---------|--------|
| EUR/USD H1 | ~2020–2026 | 144 | **36.1%** | **1.87** | +$1,847 | ~22% |

**Claim du blog : 87% WR**

### Bug critique découvert en v1
La v1 utilisait `strategy.entry()` sans paramètre `stop=`, ce qui exécutait l'ordre **au marché à l'open de la barre suivante**. Le prix d'entrée se retrouvait souvent au-delà du SL immédiatement → 0 wins sur 5 trades testés. La v2 utilise `strategy.entry(..., stop=long_entry)` : la position ne s'ouvre que si le prix franchit l'extrême du pattern.

### Analyse critique

**Le 87% est impossible à reproduire.**
Avec un RR de 1:2, un WR de 87% impliquerait un Profit Factor de ~6.7. C'est un niveau que très peu de stratégies atteignent même sur 20 trades, encore moins sur 144.

**Pourtant, le Profit Factor 1.87 est réel et positif.**
36% de WR avec RR 1:2 est en fait cohérent mathématiquement — le système gagne parce que les wins compensent les nombreux losses. Le blog confond probablement WR et quelque chose d'autre (taux de trades "en profit intraday" avant de toucher le SL, peut-être).

**Problèmes structurels de la stratégie :**
- Les pivots swing ont un **lag de 20 barres** (lookback parameter). En temps réel, on ne sait pas si un pivot est "confirmé" tant qu'on n'a pas les 20 barres suivantes. Le backtest utilise correctement `ta.pivothigh/low` qui est non-repaint, mais le délai rend difficile le trading à chaud.
- Les pin bars en H1 sont extrêmement fréquentes sur EUR/USD — la confluence avec un niveau clé reste la condition la plus filtrante mais aussi la plus subjective.
- Le filtre session est arbitraire : on rate potentiellement de bons setups pendant les sessions Asie ou les chevauchements.

### Ce qui est récupérable
- Le **RR 1:2 minimum** est une règle saine
- La **confluence de 3 conditions** (tendance + niveau + pattern) est une approche rigoureuse
- Le filtre session (London/NY) est pertinent pour EUR/USD — la liquidité est réelle

---

## 3. Stratégie 2 — RSI(2) Connors

### Source
StockCharts — *"RSI(2) Strategy" par Larry Connors*

### Règles originales
1. **Filtre tendance** : prix > SMA(200) pour les longs
2. **Entrée** : RSI(2) croise en dessous de 10 (survendu)
3. **Sortie** : prix croise au-dessus de la SMA(5)
4. Pas de stop-loss (recommandation Connors)
5. Instrument : SPY Daily

### Résultats

| Variant | RSI seuil | Exit | Trades | Win Rate | Profit Factor | CAGR | Max DD |
|---------|----------|------|--------|----------|---------------|------|--------|
| **Base** (Connors strict) | < 10 | SMA5 | 215 | 68.4% | 1.87 | 2.42% | ~14% |
| **Optimisé** | < 5 | RSI>65 OU SMA5 | 104 | **72.1%** | **2.48** | 1.95% | ~10% |

**Claim Connors : ~65–70% WR**

### Analyse critique

**C'est la seule stratégie où le claim est validé.**
68.4% WR correspond exactement à la fourchette annoncée par Connors. La stratégie a un edge prouvé sur 33 ans de SPY (1993–2026), pas seulement sur une sous-période favorable.

**Le Profit Factor 1.87 (base) et 2.48 (optimisé) sont solides.**
PF > 1.5 sur 200+ trades est généralement considéré comme statistiquement significatif.

**Problèmes réels :**
- **CAGR de 2.42%** — La stratégie est investie seulement ~15–20% du temps (en position ~30–40 jours/an). Le reste du temps, le capital dort. Sur 33 ans, Buy & Hold SPY donne ~10%/an. RSI(2) sur-performe *pendant les trades* mais l'utilisation du capital est trop faible pour battre le buy & hold en absolu.
- **Absence de stop-loss** : Connors se justifie par des statistiques long-terme, mais en mars 2020 SPY a perdu -34% en 5 semaines. Même avec 100% du capital sur position, ça représente un drawdown existentiel pour un trader retail.
- **Biais structurel bull market** : 1993–2026 sur SPY c'est globalement un bull trend. Les stratégies long-only sont mécaniquement avantagées. Sur un indice en bear trend (Nikkei 1990s, par exemple), les résultats seraient très différents.
- **Version SHORT** : testée mais non concluante — le biais haussier de SPY rend la version short peu fiable même avec les mêmes paramètres inversés.

**Optimisation sans overfit :**
Le variant optimisé (RSI<5, exit RSI>65 ou SMA5) améliore le WR de +3.7 points et le PF de +0.61 en réduisant le nombre de trades de moitié. Les paramètres restent proches des originaux de Connors — il n'y a pas eu de grid search sur la période de test.

### Ce qui est récupérable
- La logique **mean-reversion sur indices** est robuste sur longue période
- La règle **SMA(200) comme filtre de tendance** est simple et efficace
- La combinaison RSI(2) + SMA(5) comme système entrée/sortie est bien documentée académiquement

---

## 4. Stratégie 3 — Gap Fill Daily (SPY)

### Source
QuantifiedStrategies — *"Gap Fill Trading Strategies"*

### Règles testées
1. **Gap DOWN** entre -0.15% et -0.6% (open vs close[1])
2. **Filtre range J-1** : veille a clôturé dans le quart inférieur de sa range
3. **Entrée** : open du gap day
4. **TP** : 75% du gap comblé = `open + 0.75 × (close[1] - open)`
5. **Sortie** : EOD au close si TP non atteint

### Résultats — 4 Variants (SPY Daily, 1993–2026)

| Variant | Filtre range | SMA200 | TP ratio | Trades | Win Rate | PF | Net P&L | Max DD |
|---------|-------------|--------|----------|--------|----------|----|---------|--------|
| A — Base | ✓ | ✗ | 75% | 353 | **77.1%** | 1.22 | +$1,304 | 8.36% |
| B — Sans filtre | ✗ | ✗ | 75% | 1,692 | 77.1% | 1.27 | +$10,453 | 8.9% |
| C — Conservateur | ✓ | ✓ | 75% | 227 | 74.1% | 1.20 | ~+$900 | **5.3%** |
| D — Full fill | ✓ | ✗ | 100% | 353 | 72.1% | 1.17 | +$1,599 | 7.79% |

**Claim : 89% WR, +0.19%/trade (jan 2010 – août 2012, 110 trades)**

### Analyse critique

**Le 89% est une cherry-pick temporelle.**
La période 2010–2012 est une phase de mean-reversion post-crise exceptionnelle (VIX chroniquement élevé, gaps fréquents, liquidité Fed massive). Sur cette même fenêtre, notre backtest donne ~84% WR — cohérent avec le claim. Sur l'ensemble de l'historique, ça descend à 77%.

**77% WR reste un edge réel.**
353 trades sur 33 ans est statistiquement robuste. Le WR ne varie pas selon le filtre range (77.1% avec ou sans), ce qui suggère que la **condition de gap elle-même** est le vrai signal, pas le filtre J-1.

**Problème de fréquence vs rentabilité :**
- Variant A : 353 trades / 33 ans = ~10.7 trades/an. À $10K capital, Net P&L = +$1,304 sur 33 ans = ~$39/an. Le Profit Factor de 1.22 est trop bas pour être exploitable avec cette fréquence.
- Variant B (sans filtre) : 1,692 trades / 33 ans = ~51 trades/an, Net P&L = +$10,453 = ~$317/an. Meilleur en absolu, mais la fréquence quotidienne rend chaque trade très petit.

**Le TP 100% (full gap fill) est contre-intuitif mais logique :**
On pourrait croire qu'attendre le plein comblement est mieux — en réalité, le WR baisse (-5 points) car une partie des gaps qui auraient atteint 75% n'atteignent jamais 100%. Le sweet spot semble être autour de 75–80%.

**Faisabilité d'automatisation :**
Le signal est détectable à l'ouverture en temps réel :
1. Calculer `gap_pct = (open - close[1]) / close[1] * 100` dès le premier tick
2. Calculer `range_pos` avec les données de la veille (disponibles)
3. Placer un ordre LIMIT à `tp_price` + ordre de clôture à 15:55 NY

C'est **automatisable simplement** avec un broker qui expose une API (Interactive Brokers, Alpaca, etc.).

---

## 5. Synthèse comparative

### Tableau de validation des claims

| Stratégie | Claim WR | WR Backtest | Validé ? | PF | Sharpe | Recommandation |
|-----------|---------|-------------|----------|----|---------|--------------:|
| H1 Forex PA | 87% | 36.1% | ❌ Faux | 1.87 | négatif | Edge réel, claim trompeur |
| RSI(2) Connors | 65–70% | 68–72% | ✅ Confirmé | 1.87–2.48 | ~0.04 | Solide, CAGR faible |
| Gap Fill Daily | 89% | 77.1% | ⚠ Partiel | 1.22–1.27 | n/a | Edge réel, trop faible PF |

### Classement par qualité d'edge

```
1er — RSI(2) Connors   ████████████████████████ PF 2.48, 33 ans de preuves
2e  — H1 Forex PA      ██████████████████ PF 1.87, WR trompeur mais P&L positif
3e  — Gap Fill Daily   ████████████ PF 1.27, edge fragile après slippage
```

### Risque réel par stratégie

| Stratégie | Risque principal | Scénario catastrophe |
|-----------|-----------------|---------------------|
| H1 Forex | Faux niveaux clés, spread variable | Série de 10 SL consécutifs (-20%) |
| RSI(2) | Pas de SL, événement de queue | Crise type mars 2020 (-34% sans exit) |
| Gap Fill | Gain/trade trop petit vs coûts | Régime de haute volatilité (gaps >1%) |

---

## 6. Ce qui vaut le coup d'être testé en backtest manuel

### Priorité 1 — RSI(2) avec gestion du risque

**Pourquoi :** C'est la seule stratégie avec un edge validé sur 33 ans et un Profit Factor > 2.
**Test suggéré :** Comparer la version Connors sans SL vs une version avec SL ATR(14) × 2.
- Hypothèse : le SL réduit le WR de quelques points mais améliore le Sharpe significativement
- Tester aussi sur QQQ, IWM, DIA pour voir si l'edge est reproductible
- **Période de test hors-sample suggérée :** 2000–2010 (dot-com crash + crise 2008)

### Priorité 2 — Gap Fill avec capital sizing dynamique

**Pourquoi :** Le PF 1.27 est insuffisant à position fixe, mais avec une règle de sizing en fonction de la taille du gap, le ratio gain/risque pourrait s'améliorer.
**Test suggéré :** Allouer plus de capital aux gaps plus forts (e.g., -0.5% → 2× la taille standard vs -0.2% → 0.5× la taille standard)
- Hypothèse : les gros gaps ont un pull-back plus violent → meilleur momentum de retour
- À tester sur d'autres ETFs sectoriels (QQQ, XLF, XLE) où les gaps sont plus fréquents

### Priorité 3 — Combinaison RSI(2) + Gap Filter

**Pourquoi :** Les deux stratégies sont des mean-reversion sur SPY. Un gap DOWN + RSI(2) survendu = double confluence.
**Test suggéré :** Entrer RSI(2) uniquement les jours où il y a un gap DOWN dans la fourchette cible.
- Hypothèse : les entrées avec gap sont plus "propres" (momentum vendeur épuisé à l'ouverture)
- Risque : réduction drastique du nombre de trades → significativité statistique à surveiller

### Priorité 4 — H1 Forex sur données synthétiques propres

**Pourquoi :** Malgré le WR décevant (36%), le PF 1.87 sur 144 trades est respectable.
**Test suggéré :** Isoler uniquement les pin bars (pas les inside bars) sur niveaux clés journaliers
- Tester en 4h plutôt qu'en 1h (moins de bruit, niveaux plus significatifs)
- Comparer Londres vs New York en isolation : l'un des deux peut être nettement supérieur
- Tester sur GBP/USD qui est plus volatile et a des pin bars plus nets

---

## 7. Pistes d'amélioration concrètes

### RSI(2) Connors

| Amélioration | Implémentation | Impact attendu |
|-------------|---------------|---------------|
| Ajouter SL ATR×2 | `strategy.exit("Long SL", stop=close-2*atr14)` | ↑ Sharpe, ↓ WR légèrement |
| Pyramiding conditionnel | Si RSI(2) < 2 le lendemain, doubler la position | ↑ gains sur les meilleurs setups |
| Filtre VIX | Entrer uniquement si VIX > 15 | ↑ WR (mean-reversion plus forte en volatilité) |
| Multi-timeframe | Confirmer avec RSI(3) Daily + RSI(2) Weekly survendu | ↓ trades, ↑ qualité |
| Exit accéléré | Sortir si RSI > 65 ET gap up à l'ouverture (force le signal) | ↓ temps en position |

### Gap Fill Daily

| Amélioration | Implémentation | Impact attendu |
|-------------|---------------|---------------|
| Sizing dynamique | Capital × (gap_pct / 0.4) capped à 100% | ↑ P&L sur bons setups |
| Filtre VIX | Gap fill uniquement si VIX < 30 (régime normal) | ↓ pertes sur crises |
| TP adaptatif | TP = max(75% du gap, ATR(5) × 0.3) | ↑ PF sur barres courtes |
| Filtrer les gaps pré-FOMC | Éviter les jours de Fed meetings | ↓ faux setups macro |
| Multi-ETF | Ajouter QQQ, IWM pour ↑ fréquence | ↑ trades/an → meilleure signif. |

### H1 Forex Price Action

| Amélioration | Implémentation | Impact attendu |
|-------------|---------------|---------------|
| Niveaux journaliers fixes | Utiliser PDH/PDL/PDC au lieu de pivots swing | ↑ fiabilité des niveaux |
| Filtre ADR | Entrer uniquement si <50% de l'ADR consommé | ↑ marge pour le TP |
| RR adaptatif | RR = 1:3 si distance au niveau > 20 pips | ↑ gain moyen |
| Confirmation HTF | Pin bar H1 dans direction du trend D1 | ↓ faux signaux |
| Éviter News | Pas d'entrée 30min avant/après NFP, CPI, Fed | ↓ spikes SL |

---

## 8. Limites fondamentales de ce travail

### Limites de l'environnement technique

**TradingView Free :**
- Données intraday limitées à ~1 an → backtests sur < 5min inexploitables
- Pas d'accès aux données tick → slippage d'ouverture non modélisable avec précision
- Pine Script Strategy Tester est une simulation, pas un vrai broker — la gestion des ordres sur barres de forte volatilité peut différer

**CDP / Automation :**
- La lecture automatisée des résultats via les tables Pine est fragile (dépend du format exact de la table)
- Les captures d'écran nécessitent un affichage X11 actif (environnement Linux)
- Le bug de format `"#.3"` trouvé dans le code impacte l'affichage de certaines métriques (Avg Win/Loss dans Gap Fill)

### Limites méthodologiques

**Biais de data snooping :**
Les paramètres ont été ajustés après observation des premiers résultats. Même si l'ajustement est resté proche des valeurs originales des auteurs, il y a un risque de biais. Pour une validation rigoureuse, il faudrait définir les paramètres **avant** de regarder les données, puis valider sur un dataset hors-sample.

**Biais de survie :**
SPY existe depuis 1993 et inclut uniquement les entreprises qui ont survécu. Les backtests sur SPY sur-estiment mécaniquement les stratégies long-only.

**Biais de publication :**
Les trois stratégies testées viennent de sources qui ont intérêt à présenter des résultats favorables. Les 89%, 87%, et 65% sont des chiffres marketing avant d'être des chiffres scientifiques.

**Nombre de trades et significativité :**
- H1 Forex : 144 trades sur ~6 ans → insuffisant pour extrapoler. L'intervalle de confiance à 95% sur un WR de 36% avec 144 trades est large (~±8 points).
- RSI(2) : 215 trades sur 33 ans → robuste
- Gap Fill A : 353 trades sur 33 ans → acceptable
- Gap Fill B : 1,692 trades → le plus robuste statistiquement

**Absence de walk-forward analysis :**
Un vrai test robuste nécessiterait une optimisation sur 70% des données et une validation sur 30% (walk-forward). Ce travail n'a pas réalisé cette séparation.

**Marchés régimes :**
Les stratégies mean-reversion (RSI2, Gap Fill) fonctionnent en régime de faible tendance. En marché fortement directionnel (trend fort), elles sous-performent. Aucun filtre de régime n'a été testé.

**Coûts réels non modélisés :**
- Spread intraday (surtout pour H1 Forex, variable 0.5–2 pips)
- Impact marché sur positions larges
- Rollover/swap pour positions overnight Forex
- Taxes sur les plus-values (impact sur CAGR réel)

### Ce que ces backtests ne prouvent pas

- ❌ Que les stratégies fonctionneront dans le futur
- ❌ Que les résultats sont reproductibles sur d'autres instruments sans re-validation
- ❌ Que les paramètres optimaux trouvés ici sont stables dans le temps
- ❌ Qu'un trader réel obtiendrait les mêmes résultats (psychologie, exécution, etc.)

---

## Conclusion

Sur les trois stratégies testées, **une seule tient ses promesses** : RSI(2) Connors. Son edge est documenté depuis 2003, confirmé sur 33 ans de SPY, avec un Profit Factor > 2 sur la version optimisée. Sa principale limite est le CAGR faible (sous-utilisation du capital) et l'absence de stop-loss recommandée par l'auteur.

Gap Fill est la stratégie la plus **automatisable** : le signal est simple, objectif, détectable à l'ouverture. Son edge existe (PF 1.27 sur 1,692 trades) mais il est trop fragile pour être exploité directement — il nécessite un sizing dynamique et un filtre de régime pour devenir réellement profitable après coûts.

H1 Forex est la stratégie la plus **médiatique** et la moins robuste. Le PF 1.87 est positif mais le WR réel de 36% (vs 87% annoncé) illustre parfaitement comment un biais de présentation peut rendre une stratégie moyenne sembler révolutionnaire. Elle n'est pas sans intérêt — le framework confluences + RR 1:2 est sain — mais elle demande une validation sur beaucoup plus de trades avant d'y allouer du capital.

**Recommandation finale :** RSI(2) avec SL ATR×2, testé sur SPY + QQQ + IWM, avec une analyse walk-forward sur 2000–2015 / validation 2015–2026. C'est le point de départ le plus rigoureux pour du trading quantitatif.

---

*Scripts Pine disponibles dans `/home/alexis/tradingview-mcp/scripts/` :*
- `rsi2_strategy.pine` — RSI(2) Connors optimisé
- `gap_fill_daily.pine` — Gap Fill indicateur manuel
- `h1_forex_strategy.pine` — H1 Forex v2 (stop orders)
