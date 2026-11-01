# H2 — FA Q8 tensor-core rows==4, la « mission Kolbo » (8-16/10)

**Verdict : NO-GO définitif (18/10) — surpassé par T3, désactivé en prod.**

## Chronologie (le GO, puis le NO-GO)

H2 est la mission « Kolbo » du 8-16/10 : un kernel FA Q8 tensor-core dédié à la
géométrie rows==4 (le cas exact du verify MTP n_max=3 → 4 tokens), sur la base
prod de l'époque (H2 seul en prod, `.so` a4b6f440).

| date | fait |
|---|---|
| 8-9/10 | Protocole de capture nsys, root cause « ordre CUDA ≠ ordre nvidia-smi », gates parité prod (`PROTOCOLE.md`) |
| 13/10 | run7 CUPTI v2 : verdict instrumentale (union/chevauchement streams 16/17, idle ≤ 10,8 %) |
| 14-15/10 | A/B applicatifs run8-9c3 : **GO** — H2 vs prod +11 à +14 % @107k, +2,1 % @41k (dans le bruit) ; H2 déployée en production expérimentale |
| 15/10 | H2b (variante bit-identique, 6 sites) : le gain vient du **partitionnement** BC=8, pas de la tuile — H2b = prod (+0,6 %) |
| 18/10 | T3 validé et déployé (voir `opti-t3-volta-small`). T3 couvre rows 1-8, dont rows==4 : **le chemin H2 est remplacé** |
| 18/10 | Bench de conflit (MTP ON, `.so` T3 prod d2df07b8) : H2 ON vs H2 OFF = **−15 à −17 % avec H2 ON** → T3 bat H2 même sur rows==4 → **H2 désactivé** (`GGML_CUDA_VOLTA_Q8_FATTN_TC=0`) |

**Le NO-GO tient à deux mesures complémentaires** :
1. **Applicative** (18/10, MTP ON, `.so` T3) : +17,4 % @114k / +15,5 % @160k en
désactivant H2 (T3 gagne partout, pas de conflit de dispatch).
2. **Kernel pur** (microbench rows 1-8, dispatch prod) : à T=4, stock 2194 µs,
**H2 1532 µs, T3 669 µs** → T3 est +56,3 % plus rapide que H2 sur la géométrie
dont H2 était le spécialiste.

## Contenu

```
# archive brute de la mission (383 fichiers) :
ETAT-raw.md    # journal complet (32 sections) + ETAT-*.md (verdicts par jour)
PROTOCOLE.md   # invariants, pièges CUDA/nsys · KOLBO-ncu-volta-20261013.md
e2-src/        # sources (cu/cuh/rsp/build + logs) · nightrun8/ (A/B run8→run9c3)
ab_results_*/  # réponses gen + mesures · nmax_runs/ · perf-n3/ · sync_runs/
patch-h2-*.patch  # le patch H2 (2 lignes) ; H2b (6 sites) · cupath_shim*.c
docs/protocols/   # 02-AB-ref-h2.md (protocole A/B de référence), 03-build-lib-h2.md
bench/           # les 7 scripts A/B (run8→run9c3 + analyse) — extraits de nightrun8/
bigfiles/        # manifest sha256 (hors git : libs H2/H2b + binaires microbench, 273 Mo)
```

## `.so` / binaires associés (manifest `bigfiles/bigfiles.sha256`)

| md5 | état |
|---|---|
| `a4b6f440` | **H2 seul — la prod de l'époque** (backup rollback conservé) |
| `0eb5afbd` | H2b (bit-identique, 6 sites) — sans gain |
| `d2df07b8` | prod T3 (le successeur ; H2 en gate ON par défaut, désactivé par env) |

Les binaires sont recompilables depuis `e2-src/` + patches sur la machine de
référence (`docs/protocols/03`), ou récupérables du canal séparé avec le manifest.

## Re-validation

Le bench qui a tranché (NO-GO) est **re-jouable** : A/B MTP ON, `.so` T3 prod,
seul le gate H2 varie (voir `opti-t3-volta-small` §Re-validation, table H2 :
53,93 vs 45,90 t/s @114k, 44,59 vs 36,28 @160k).

L'A/B **GO** (run9c3, H2 vs REF @107k, 6 essais interleave, session isolée) :
`bench/run9c3_bench.py` (~25 min) → comparer à `nightrun8/ab_run9c3/results.jsonl`
(eval_ms REF ≈ 40,3/40,6/41,9 s · H2 ≈ 35,3/35,2/36,7 s, gen 6/6 sauvegardés).

## Caveats (honnêteté de la mesure)

- Tous les A/B sont sur **prompt filler répéteur** (acceptance MTP ~0,8-1,0 vs
  ~0,6-0,8 en prod réelle) ; le gain est mesuré en ms/tok (indépendant du contenu).
- Le gain de H2 est **indissociable de la divergence textuelle** (associativité
  fp32, 1 ULP ; évaluation qualité : aucune régression macroscopique) — c'est ce
  qui a rendu le GO « production expérimentale » et non « prod définitive ».
- H2 @41k : +2,1 % = dans le bruit (écart intra-bras ~1 %) — le gain réel n'existe
  qu'au-dessus du seuil n_kv ≈ 107 521 (régime du partitionnement BC=8).
- Certaines mesures H2-PROD (14/10) sur V100 partagée sont marquées « entachées »
  (`ETAT-raw.md` §28.4) ; les A/B de référence (run9b/c/c2/c3) sont en session
  isolée (V100 < 1000 MiB avant chaque essai, vérifié dans les logs).

## Lien avec la publication

Ce repo est la 7ᵉ opti de la pile [`llama-v100-opti`](https://github.com/pitouche/llama-v100-opti)
(6 GO/CLOS) + H2 (NO-GO). H2 y figure dans `OPTI.md` (§5.3/5.4/6.1) pour le
**conflit de dispatch** (le verdict NO-GO), pas pour la mission — la mission
entière (protocoles, CUPTI, A/B, divergence textuelle) vit ici, issue du kit
`v100-bench-kit` d'où elle a été extraite le 01/11/2026.
