# Test de rendu (temporaire)

CASA_italique_backtick : *texte $`x_i + y_j`$ fin*

CASB_gras_backtick : **texte $`x_i + y_j`$ fin**

CASC_plain_backtick : texte $`x_i + y_j`$ fin

CASD_italique_plain : *texte $x_i + y_j$ fin*

CASE_gras_plain : **texte $x_i + y_j$ fin**

CASF_inline_double : texte $$x_i + y_j$$ fin

CASG_emphase_partielle : texte *en italique* puis $`x_i + y_j`$ hors italique

Table avec backtick :

| col | valeur |
|---|---|
| TABA | $`\alpha_i,\beta_j`$ |

Table avec plain :

| col | valeur |
|---|---|
| TABB | $\alpha_i,\beta_j$ |

Table avec double :

| col | valeur |
|---|---|
| TABC | $$\alpha_i,\beta_j$$ |
