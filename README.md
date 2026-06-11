#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Mappa coropletica dei collegi uninominali della Camera (400 collegi)
Confini ufficiali Eurostat-GISCO NUTS 2024 (file locale, EPSG:3035)
Lettura diretta del GeoPackage via sqlite3 + parser WKB puro Python.
"""
import sqlite3, struct, json
import matplotlib
matplotlib.use('Agg')
import matplotlib.pyplot as plt
from matplotlib.colors import LinearSegmentedColormap, Normalize
from matplotlib.patches import Polygon as MplPolygon
from matplotlib.collections import PatchCollection
import matplotlib.cm as cm

GPKG = '/mnt/user-data/uploads/NUTS_RG_20M_2024_3035.gpkg'

COLLEGI_CAMERA = json.load(open('/home/claude/riparto_392.json'))

# ---------------------------------------------------------------- WKB parser
def parse_gpkg_blob(blob):
    """Estrae il WKB dal GeoPackage Binary e restituisce lista di anelli
    esterni [(x,y), ...] per ogni poligono."""
    assert blob[0:2] == b'GP'
    flags = blob[3]
    env_ind = (flags >> 1) & 0b111
    env_sizes = {0: 0, 1: 32, 2: 48, 3: 48, 4: 64}
    offset = 8 + env_sizes.get(env_ind, 0)
    return parse_wkb(blob, offset)[0]

def parse_wkb(buf, off):
    bo = '<' if buf[off] == 1 else '>'
    gtype = struct.unpack_from(bo + 'I', buf, off + 1)[0] & 0xFF
    off += 5
    polys = []
    if gtype == 3:  # Polygon
        nrings = struct.unpack_from(bo + 'I', buf, off)[0]; off += 4
        rings = []
        for _ in range(nrings):
            npts = struct.unpack_from(bo + 'I', buf, off)[0]; off += 4
            pts = struct.unpack_from(bo + f'{npts*2}d', buf, off)
            off += npts * 16
            rings.append(list(zip(pts[0::2], pts[1::2])))
        polys.append(rings)
    elif gtype == 6:  # MultiPolygon
        npolys = struct.unpack_from(bo + 'I', buf, off)[0]; off += 4
        for _ in range(npolys):
            sub, off = parse_wkb(buf, off)
            polys.extend(sub)
    else:
        raise ValueError(f'Tipo geometria non gestito: {gtype}')
    return polys, off

# ---------------------------------------------------------------- lettura dati
con = sqlite3.connect(GPKG)
cur = con.cursor()
tbl = cur.execute("SELECT table_name FROM gpkg_contents WHERE data_type='features'").fetchone()[0]
cols = [c[1] for c in cur.execute(f'PRAGMA table_info({tbl})').fetchall()]
geom_col = cur.execute(f"SELECT column_name FROM gpkg_geometry_columns WHERE table_name='{tbl}'").fetchone()[0]
print('Tabella:', tbl, '| colonne:', cols)

rows = cur.execute(
    f"SELECT NUTS_ID, NAME_LATN, {geom_col} FROM {tbl} "
    f"WHERE LEVL_CODE=2 AND NUTS_ID LIKE 'IT%'").fetchall()
con.close()
print(f'Regioni italiane NUTS-2 trovate: {len(rows)}')

features = []
for nuts_id, name, blob in rows:
    polys = parse_gpkg_blob(blob)
    features.append((nuts_id, name, polys))

# ---------------------------------------------------------------- resa grafica
fig, ax = plt.subplots(figsize=(9, 11), dpi=300)
fig.patch.set_facecolor('#fbfbfb')
ax.set_facecolor('#eef4fb')

cmap = LinearSegmentedColormap.from_list(
    'blues_ist', ['#dbeafe', '#93c5fd', '#3b82f6', '#1e40af', '#1a365d'], N=256)
vmin, vmax = 1, 66
norm = Normalize(vmin=vmin, vmax=vmax)

def centroid_largest(polys):
    """Centroide dell'anello esterno del poligono più esteso (area di Gauss)."""
    best, best_area = None, -1
    for rings in polys:
        ext = rings[0]
        a = cx = cy = 0.0
        for (x1, y1), (x2, y2) in zip(ext, ext[1:] + ext[:1]):
            cross = x1 * y2 - x2 * y1
            a += cross; cx += (x1 + x2) * cross; cy += (y1 + y2) * cross
        if a == 0:
            continue
        area = abs(a / 2)
        if area > best_area:
            best_area = area
            best = (cx / (3 * a), cy / (3 * a))
    return best

for nuts_id, name, polys in features:
    val = COLLEGI_CAMERA.get(nuts_id)
    face = cmap(norm(val)) if val else '#dddddd'
    patches = [MplPolygon(rings[0], closed=True) for rings in polys]
    pc = PatchCollection(patches, facecolor=face, edgecolor='#c9a227', linewidth=0.8)
    ax.add_collection(pc)

# Etichette dopo i poligoni, così restano in primo piano
LABEL_OFFSET = {  # piccoli aggiustamenti manuali (metri, EPSG:3035)
    'ITC3': (16000, -26000),   # Liguria, regione stretta ad arco
    'ITF2': (8000, 0),     # Molise
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

ax.autoscale_view()
ax.set_aspect('equal')
ax.set_axis_off()

ax.set_title("DISTRIBUZIONE GEOGRAFICA DEI COLLEGI UNINOMINALI\n"
             "Camera dei Deputati — 392 collegi nazionali + 8 Estero = 400 (art. 56 Cost.)",
             fontsize=13, fontweight='bold', color='#1a365d', pad=18)

sm = cm.ScalarMappable(cmap=cmap, norm=norm)
sm.set_array([])
cbar = fig.colorbar(sm, ax=ax, fraction=0.03, pad=0.02, shrink=0.55)
cbar.set_label('N. Collegi per Circoscrizione', fontsize=8, color='#1a365d')
cbar.ax.tick_params(labelsize=7)

fig.text(0.5, 0.02,
         "Riparto ex art. 56, c. 4, Cost.: quozienti interi e più alti resti — Popolazione legale Censimento ISTAT 2021\n"
         "Confini Eurostat-GISCO NUTS 2024, EPSG:3035 (CC-BY) — Regola aurea ±15% — Deroga aree interne −30%",
         ha='center', fontsize=7.5, color='#4a5568', style='italic')

plt.tight_layout(rect=[0, 0.035, 1, 0.97])
plt.savefig('/home/claude/mappa_collegi_REALE.png', dpi=300,
            bbox_inches='tight', facecolor='#fbfbfb')
print('Fatto: mappa_collegi_REALE.png')
