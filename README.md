#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
RIPARTO COSTITUZIONALE DEI COLLEGI UNINOMINALI — Camera dei Deputati
Art. 56, quarto comma, Cost.: ripartizione dei seggi tra le circoscrizioni
"in proporzione alla popolazione legale", "sulla base dei quozienti interi
e dei più alti resti" (metodo Hare-Niemeyer).

Deputati totali: 400, di cui 8 eletti nella Circoscrizione Estero
(art. 56, secondo comma) → seggi da ripartire sul territorio nazionale: 392.

Popolazione di riferimento: popolazione legale, Censimento permanente
ISTAT 2021 (da verificare con i valori pubblicati in Gazzetta Ufficiale).
"""

SEGGI_NAZIONALI = 392  # 400 - 8 (Circoscrizione Estero)

# Popolazione legale Censimento permanente 2021 (ISTAT)
POPOLAZIONE = {
    'ITC1': ('Piemonte',              4_256_350),
    'ITC2': ("Valle d'Aosta",           123_360),
    'ITC3': ('Liguria',               1_509_805),
    'ITC4': ('Lombardia',             9_943_004),
    'ITH1': ('Bolzano/Bozen',           532_644),
    'ITH2': ('Trento',                  540_958),
    'ITH3': ('Veneto',                4_847_745),
    'ITH4': ('Friuli-Venezia Giulia', 1_194_647),
    'ITH5': ('Emilia-Romagna',        4_425_366),
    'ITI1': ('Toscana',               3_661_981),
    'ITI2': ('Umbria',                  854_137),
    'ITI3': ('Marche',                1_484_427),
    'ITI4': ('Lazio',                 5_714_882),
    'ITF1': ('Abruzzo',               1_272_973),
    'ITF2': ('Molise',                  290_636),
    'ITF3': ('Campania',              5_590_681),
    'ITF4': ('Puglia',                3_890_250),
    'ITF5': ('Basilicata',              537_577),
    'ITF6': ('Calabria',              1_841_300),
    'ITG1': ('Sicilia',               4_833_705),
    'ITG2': ('Sardegna',              1_575_028),
}

pop_tot = sum(p for _, p in POPOLAZIONE.values())
quoziente = pop_tot / SEGGI_NAZIONALI  # quoziente naturale (Hare)

print(f'Popolazione legale totale : {pop_tot:>12,}'.replace(',', '.'))
print(f'Seggi nazionali           : {SEGGI_NAZIONALI}')
print(f'Quoziente naturale        : {quoziente:>12,.3f}'.replace(',', '.'))
print()

# Fase 1: quozienti interi
righe = []
for nuts, (nome, pop) in POPOLAZIONE.items():
    q = pop / quoziente
    intero = int(q)
    resto = q - intero
    righe.append([nuts, nome, pop, q, intero, resto, 0])

assegnati = sum(r[4] for r in righe)
residui = SEGGI_NAZIONALI - assegnati
print(f'Seggi per quoziente intero: {assegnati}  |  Seggi residui: {residui}')

# Fase 2: più alti resti
for r in sorted(righe, key=lambda x: -x[5])[:residui]:
    r[6] = 1

print()
print(f"{'NUTS':<6}{'Circoscrizione':<24}{'Popolazione':>12}{'Quoz.':>10}"
      f"{'Interi':>8}{'Resto':>8}{'+R':>4}{'TOTALE':>8}")
print('-' * 80)
totale = 0
RIPARTO = {}
for r in sorted(righe, key=lambda x: x[0]):
    tot = r[4] + r[6]
    totale += tot
    RIPARTO[r[0]] = tot
    print(f"{r[0]:<6}{r[1]:<24}{r[2]:>12,}{r[3]:>10.3f}"
          f"{r[4]:>8}{r[5]:>8.3f}{'+1' if r[6] else '':>4}{tot:>8}".replace(',', '.'))
print('-' * 80)
print(f"{'TOTALE NAZIONALE':<42}{'':>10}{'':>8}{'':>8}{'':>4}{totale:>8}")
print(f"{'Circoscrizione Estero (art. 56, c. 2)':<72}{8:>8}")
print(f"{'TOTALE CAMERA DEI DEPUTATI':<72}{totale + 8:>8}")

import json
with open('/home/claude/riparto_392.json', 'w') as f:
    json.dump(RIPARTO, f)
