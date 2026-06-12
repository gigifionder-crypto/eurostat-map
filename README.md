#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
================================================================================
RIPARTO COSTITUZIONALE E MAPPA DEI COLLEGI UNINOMINALI
Camera dei Deputati — 392 collegi nazionali + 8 Circoscrizione Estero = 400
================================================================================

Script unificato e autosufficiente. Esegue, nell'ordine:

  1. il riparto dei 392 seggi nazionali tra le circoscrizioni regionali,
     ai sensi dell'art. 56, quarto comma, della Costituzione: proporzione
     alla popolazione legale, quozienti interi e più alti resti
     (metodo Hare-Niemeyer);

  2. la resa cartografica coropletica sui confini ufficiali Eurostat-GISCO
     NUTS 2024 (EPSG:3035), letti direttamente dal GeoPackage tramite
     sqlite3 e un parser WKB in Python puro — nessuna dipendenza da
     geopandas, fiona o shapely.

REQUISITI:  solo matplotlib (pip install matplotlib)

ESECUZIONE:
    python3 riparto_e_mappa_collegi.py [percorso/NUTS_RG_20M_2024_3035.gpkg]

Se il percorso non è indicato, il GeoPackage è cercato nella cartella
corrente. Il file è scaricabile da:
https://gisco-services.ec.europa.eu/distribution/v2/nuts/gpkg/NUTS_RG_20M_2024_3035.gpkg

OUTPUT:
    riparto_art56_tabella.txt   — tabella completa del riparto
    mappa_collegi_REALE.png     — mappa coropletica, 300 dpi
================================================================================
"""
import sys, os, sqlite3, struct
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
from matplotlib.colors import LinearSegmentedColormap, Normalize
from matplotlib.patches import Polygon as MplPolygon
from matplotlib.collections import PatchCollection
import matplotlib.cm as cm

GPKG = sys.argv[1] if len(sys.argv) > 1 else 'NUTS_RG_20M_2024_3035.gpkg'
if not os.path.exists(GPKG):
    sys.exit(f"GeoPackage non trovato: {GPKG}\n"
             "Indicare il percorso come primo argomento o scaricare il file da "
             "https://gisco-services.ec.europa.eu/distribution/v2/nuts/gpkg/")

# ============================================================================
# PARTE I — RIPARTO EX ART. 56, QUARTO COMMA, COST.
# ============================================================================
SEGGI_NAZIONALI = 392   # 400 deputati − 8 Circoscrizione Estero (art. 56, c. 2)

# Popolazione legale, Censimento permanente ISTAT 2021
# (valori da collazionare con la pubblicazione in Gazzetta Ufficiale)
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

pop_tot   = sum(p for _, p in POPOLAZIONE.values())
quoziente = pop_tot / SEGGI_NAZIONALI          # quoziente naturale (Hare)

righe = []
for nuts, (nome, pop) in POPOLAZIONE.items():
    q = pop / quoziente
    righe.append([nuts, nome, pop, q, int(q), q - int(q), 0])

residui = SEGGI_NAZIONALI - sum(r[4] for r in righe)
for r in sorted(righe, key=lambda x: -x[5])[:residui]:
    r[6] = 1                                   # più alti resti

COLLEGI_CAMERA = {r[0]: r[4] + r[6] for r in righe}

# Tabella di riparto su file
out = []
out.append(f"Popolazione legale totale : {pop_tot:,}".replace(',', '.'))
out.append(f"Seggi nazionali           : {SEGGI_NAZIONALI}")
out.append(f"Quoziente naturale        : {quoziente:,.3f}".replace(',', '.'))
out.append("")
out.append(f"{'NUTS':<6}{'Circoscrizione':<24}{'Popolazione':>12}{'Quoz.':>10}"
           f"{'Interi':>8}{'Resto':>8}{'+R':>4}{'TOTALE':>8}")
out.append('-' * 80)
for r in sorted(righe, key=lambda x: x[0]):
    out.append(f"{r[0]:<6}{r[1]:<24}{r[2]:>12,}{r[3]:>10.3f}{r[4]:>8}"
               f"{r[5]:>8.3f}{'+1' if r[6] else '':>4}"
               f"{r[4]+r[6]:>8}".replace(',', '.'))
out.append('-' * 80)
out.append(f"{'TOTALE NAZIONALE':<72}{sum(COLLEGI_CAMERA.values()):>8}")
out.append(f"{'Circoscrizione Estero (art. 56, c. 2)':<72}{8:>8}")
out.append(f"{'TOTALE CAMERA DEI DEPUTATI':<72}{sum(COLLEGI_CAMERA.values())+8:>8}")
tabella = '\n'.join(out)
print(tabella)
with open('riparto_art56_tabella.txt', 'w') as f:
    f.write(tabella + '\n')

# ============================================================================
# PARTE II — LETTURA DEL GEOPACKAGE (sqlite3 + parser WKB puro Python)
# ============================================================================
def parse_gpkg_blob(blob):
    """Estrae il WKB dal GeoPackage Binary; restituisce, per ogni poligono,
    la lista degli anelli [(x, y), ...]."""
    assert blob[0:2] == b'GP', 'Blob GeoPackage non valido'
    env_ind = (blob[3] >> 1) & 0b111
    env_sizes = {0: 0, 1: 32, 2: 48, 3: 48, 4: 64}
    return parse_wkb(blob, 8 + env_sizes.get(env_ind, 0))[0]

def parse_wkb(buf, off):
    bo = '<' if buf[off] == 1 else '>'
    gtype = struct.unpack_from(bo + 'I', buf, off + 1)[0] & 0xFF
    off += 5
    polys = []
    if gtype == 3:                                  # Polygon
        nrings = struct.unpack_from(bo + 'I', buf, off)[0]; off += 4
        rings = []
        for _ in range(nrings):
            npts = struct.unpack_from(bo + 'I', buf, off)[0]; off += 4
            pts = struct.unpack_from(bo + f'{npts*2}d', buf, off)
            off += npts * 16
            rings.append(list(zip(pts[0::2], pts[1::2])))
        polys.append(rings)
    elif gtype == 6:                                # MultiPolygon
        npolys = struct.unpack_from(bo + 'I', buf, off)[0]; off += 4
        for _ in range(npolys):
            sub, off = parse_wkb(buf, off)
            polys.extend(sub)
    else:
        raise ValueError(f'Tipo geometria non gestito: {gtype}')
    return polys, off

con = sqlite3.connect(GPKG)
cur = con.cursor()
tbl = cur.execute("SELECT table_name FROM gpkg_contents "
                  "WHERE data_type='features'").fetchone()[0]
geom_col = cur.execute("SELECT column_name FROM gpkg_geometry_columns "
                       f"WHERE table_name='{tbl}'").fetchone()[0]
rows = cur.execute(f"SELECT NUTS_ID, NAME_LATN, {geom_col} FROM {tbl} "
                   f"WHERE LEVL_CODE=2 AND NUTS_ID LIKE 'IT%'").fetchall()
con.close()
print(f'\nRegioni italiane NUTS-2 lette dal GeoPackage: {len(rows)}')

features = [(nid, name, parse_gpkg_blob(blob)) for nid, name, blob in rows]

# ============================================================================
# PARTE III — RESA CARTOGRAFICA
# ============================================================================
def centroid_largest(polys):
    """Centroide dell'anello esterno del poligono più esteso (formula di Gauss)."""
    best, best_area = None, -1.0
    for rings in polys:
        ext = rings[0]
        a = cx = cy = 0.0
        for (x1, y1), (x2, y2) in zip(ext, ext[1:] + ext[:1]):
            cross = x1 * y2 - x2 * y1
            a += cross; cx += (x1 + x2) * cross; cy += (y1 + y2) * cross
        if a == 0:
            continue
        if abs(a / 2) > best_area:
            best_area = abs(a / 2)
            best = (cx / (3 * a), cy / (3 * a))
    return best

fig, ax = plt.subplots(figsize=(9, 11), dpi=300)
fig.patch.set_facecolor('#fbfbfb')
ax.set_facecolor('#eef4fb')

cmap = LinearSegmentedColormap.from_list(
    'blues_ist', ['#dbeafe', '#93c5fd', '#3b82f6', '#1e40af', '#1a365d'], N=256)
norm = Normalize(vmin=1, vmax=max(COLLEGI_CAMERA.values()))

for nuts_id, name, polys in features:
    val = COLLEGI_CAMERA.get(nuts_id)
    face = cmap(norm(val)) if val else '#dddddd'
    pc = PatchCollection([MplPolygon(r[0], closed=True) for r in polys],
                         facecolor=face, edgecolor='#c9a227', linewidth=0.8)
    ax.add_collection(pc)

LABEL_OFFSET = {                       # aggiustamenti manuali (metri, EPSG:3035)
    'ITC3': (16000, -26000),           # Liguria, regione stretta ad arco
    'ITF2': (8000, 0),                 # Molise
}
for nuts_id, name, polys in features:
    val = COLLEGI_CAMERA.get(nuts_id)
    if val is None:
        continue
    c = centroid_largest(polys)
    dx, dy = LABEL_OFFSET.get(nuts_id, (0, 0))
    ax.annotate(str(val), xy=(c[0] + dx, c[1] + dy), ha='center', va='center',
                fontsize=8.5, fontweight='bold',
                color='white' if val > 22 else '#1a365d')

ax.autoscale_view(); ax.set_aspect('equal'); ax.set_axis_off()
ax.set_title("DISTRIBUZIONE GEOGRAFICA DEI COLLEGI UNINOMINALI\n"
             "Camera dei Deputati — 392 collegi nazionali + 8 Estero = 400 (art. 56 Cost.)",
             fontsize=13, fontweight='bold', color='#1a365d', pad=18)

sm = cm.ScalarMappable(cmap=cmap, norm=norm); sm.set_array([])
cbar = fig.colorbar(sm, ax=ax, fraction=0.03, pad=0.02, shrink=0.55)
cbar.set_label('N. Collegi per Circoscrizione', fontsize=8, color='#1a365d')
cbar.ax.tick_params(labelsize=7)

fig.text(0.5, 0.02,
         "Riparto ex art. 56, c. 4, Cost.: quozienti interi e più alti resti — "
         "Popolazione legale Censimento ISTAT 2021\n"
         "Confini Eurostat-GISCO NUTS 2024, EPSG:3035 (CC-BY) — "
         "Regola aurea ±15% — Deroga aree interne −30%",
         ha='center', fontsize=7.5, color='#4a5568', style='italic')

plt.tight_layout(rect=[0, 0.035, 1, 0.97])
plt.savefig('mappa_collegi_REALE.png', dpi=300, bbox_inches='tight',
            facecolor='#fbfbfb')
print('Fatto: mappa_collegi_REALE.png, riparto_art56_tabella.txt')